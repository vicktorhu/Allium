# Hinge (Lid) Open/Close

The "hinge" of the Miyoo Mini Flip (MY285) is a **lid switch** backed by a hall sensor. Allium
exposes it as a synthetic button, `Key::LidClose`, so the rest of the codebase treats it like any
other input.

> The project calls it the **lid**; "hinge" is the same thing. Only the **Miyoo Mini Flip
> (MY285)** has one.

## Where it lives

`crates/common/src/platform/miyoo/evdev.rs` — this is the only place lid detection exists.

```rust
pub struct EvdevKeys {
    pub events: EventStream,
    lid_switch_poller: Option<LidSwitchPoller>,
}

impl EvdevKeys {
    pub fn new() -> Result<Self> {
        Ok(Self {
            events: Device::open("/dev/input/event0").unwrap().into_event_stream()?,
            lid_switch_poller: DefaultPlatform::has_lid().then(LidSwitchPoller::new),
        })
    }
}
```

The poller is only created when `Platform::has_lid()` is `true`, i.e. on the MY285
(`crates/common/src/platform/miyoo/mod.rs`):

```rust
fn has_lid() -> bool {
    match detect_model() {
        MiyooDeviceModel::Miyoo283 | MiyooDeviceModel::Miyoo354 => false,
        MiyooDeviceModel::Miyoo285 => true,
    }
}
```

This guard matters: the sensor path does not exist on the other models, and the reader would
panic.

## The sensor

```rust
fn read_is_lid_open() -> Result<bool> {
    let mut file = File::open("/sys/devices/soc0/soc/soc:hall-mh248/hallvalue")
        .expect("Failed to open /sys/devices/soc0/soc/soc:hall-mh248/hallvalue");
    let mut buffer = [0u8; 2];
    file.read_exact(&mut buffer)?;
    Ok(buffer[0] == b'1')
}
```

* Path: `/sys/devices/soc0/soc/soc:hall-mh248/hallvalue` (MY285 hall sensor).
* The file contains two ASCII characters; the **first** byte is the state: `'1'` = lid open,
  anything else = lid closed.

## Edge detection

The raw sysfs file is a *level*, not an event. `LidSwitchPoller` converts it into press/release
events by remembering the previous level:

```rust
struct LidSwitchPoller {
    is_lid_open: bool,
}

impl LidSwitchPoller {
    fn new() -> Self {
        let is_lid_open = read_is_lid_open().expect("Failed to read lid switch state");
        Self { is_lid_open }
    }

    fn poll(&mut self) -> Option<KeyEvent> {
        let is_lid_open = read_is_lid_open().expect("Failed to read lid switch state");
        if is_lid_open != self.is_lid_open {
            self.is_lid_open = is_lid_open;
            if is_lid_open {
                Some(KeyEvent::Released(Key::LidClose))   // lid opened  -> release
            } else {
                Some(KeyEvent::Pressed(Key::LidClose))    // lid closed  -> press
            }
        } else {
            None
        }
    }
}
```

Semantics:

|Physical action|Synthetic event|
|---|---|
|Closing the hinge|`KeyEvent::Pressed(Key::LidClose)`|
|Opening the hinge|`KeyEvent::Released(Key::LidClose)`|

So `Key::LidClose` behaves like a button that is **held while the lid is closed**.

## Polling loop

`EvdevKeys::poll()` interleaves the lid check with evdev key events:

```rust
pub async fn poll(&mut self) -> KeyEvent {
    loop {
        if let Some(lid_event) = self.lid_switch_poller.as_mut().and_then(|lid| lid.poll()) {
            info!("Lid event detected: {:?}", lid_event);
            return lid_event;
        }

        let timeout = tokio::time::timeout(Duration::from_millis(500), self.events.next_event());
        let Ok(result) = timeout.await else { continue; };   // re-check the lid
        ...
    }
}
```

Consequences:

* The hinge is checked **at least every 500 ms** (the evdev wait has a 500 ms timeout, after
  which the loop restarts and re-reads the sensor). Typical latency for a lid event: < 500 ms.
* Because the lid check happens *before* the evdev read, a lid event is delivered promptly and
  takes priority over a queued key event.
* The initial state is captured in `LidSwitchPoller::new()`, so a lid that is already closed at
  boot does **not** generate a spurious press event. Consequence: with the default
  `lid_close_action = Shutdown`, booting with the lid closed does **not** shut the device down —
  the action only fires on a close *transition*.
* Lid events are **not** subject to the stale-press filter (`MAXIMUM_FRAME_TIME`) that applies to
  evdev key events; they are delivered as soon as they are detected.

## Cross-checks against other projects

The same hardware path and the same technique are used by the projects Allium is inspired by,
which confirms the semantics above:

* **MiniUI** (`shauninman/MiniUI`, the project Allium's UI is modelled on) reads the identical
  node and uses the identical edge-detection approach:

  ```c
  #define LID_PATH "/sys/devices/soc0/soc/soc:hall-mh248/hallvalue"
  void PLAT_initLid(void) { lid.has_lid = exists(LID_PATH); }
  int PLAT_lidChanged(int* state) {
      if (lid.has_lid) {
          int lid_open = getInt(LID_PATH);      // 1 = open
          if (lid_open != lid.is_open) { lid.is_open = lid_open; ... return 1; }
      }
      return 0;
  }
  ```

* **OnionOS** on the Flip needs a community "lid support" patch to react to the lid at all;
  stock OnionOS ignores it. MiniUI shuts the device down when the lid is closed, which is
  where Allium's default `lid_close_action = Shutdown` comes from.

A known quirk in the Flip firmware (documented by other projects' changelogs) is that the lid
sensor can wrongly report *closed* around boot, which is why the initial state is captured in
`LidSwitchPoller::new()` rather than being treated as an event.

## What the daemon does with lid events

In `crates/alliumd/src/alliumd.rs`, `handle_key_event` (the branch used when Menu is not held):

```rust
KeyEvent::Pressed(Key::LidClose) => {
    #[cfg(unix)]
    match self.power_settings.lid_close_action {
        PowerButtonAction::Suspend => self.handle_suspend().await?,
        PowerButtonAction::Shutdown => self.handle_quit().await?,
        PowerButtonAction::Nothing  => {}
    }
}
```

* `lid_close_action` defaults to **`Shutdown`** (`crates/common/src/power.rs`), so closing the
  hinge powers the device off by default.
* `Suspend` → the screen goes off and the running game is frozen with `SIGSTOP`.
* `Nothing` → closing the hinge does nothing.

And inside `handle_suspend`, the *release* of `Key::LidClose` (i.e. **opening the hinge**) wakes
the device:

```rust
if matches!(key_event, KeyEvent::Released(Key::Power))
    || matches!(key_event, KeyEvent::Released(Key::LidClose)) {
    self.keys[Key::Power] = false;
    self.keys[Key::LidClose] = false;
    break;   // wake up
}
```

So with `lid_close_action = Suspend`:

* close hinge → suspend (screen off, game frozen)
* open hinge → wake up (screen on, game continues exactly where it was)
* press power → also wakes up

With `lid_close_action = Nothing`, opening the hinge still wakes a device that is suspended for
another reason (e.g. power-button suspend).

## The setting

`lid_close_action` is edited in **Settings → Power** in the launcher
(`crates/allium-launcher/src/view/settings/power.rs`). The row is only added when
`DefaultPlatform::has_lid()` is true:

```rust
if DefaultPlatform::has_lid() {
    buttons.push((
        locale.t("settings-power-lid-close-action"),
        Box::new(Select::new(
            Point::zero(),
            power_settings.lid_close_action as usize,
            vec![
                locale.t("settings-power-power-button-action-suspend"),
                locale.t("settings-power-power-button-action-shutdown"),
                locale.t("settings-power-power-button-action-nothing"),
            ],
            Alignment::Right,
        )),
    ));
}
```

The locale keys are shared with the power button row:

|Locale key|Meaning|
|---|---|
|`settings-power-lid-close-action`|Row label ("Lid Close Action")|
|`settings-power-power-button-action-suspend`|"Suspend"|
|`settings-power-power-button-action-shutdown`|"Shutdown"|
|`settings-power-power-button-action-nothing`|"Nothing"|

Because the row is conditional, the index→field mapping in `handle_key_event` is positional:
index `4` = lid close action, and it only exists on devices with a hinge.

## Simulator and mock

* The **simulator** reports `has_lid() == true` (so the setting is visible and editable) but
  **never emits `Key::LidClose` events** — hinge behaviour cannot be tested in the simulator.
* The **mock** platform reports `has_lid() == false`.

## Button-hint support for the lid

`Key::LidClose` is part of the generic button-hint system
(`crates/common/src/view/button_icon.rs`): it renders as the label `LID` and would use the asset
`button-lid.png` if it existed (it does not ship with the repo, so the vector fallback is used).
No current screen shows a LID hint — the setting page only uses A/B hints — but the plumbing
exists if a UI wants to advertise the hinge.

## Caveats / known rough edges

* `read_is_lid_open()` uses `.expect(...)` on `File::open`. If the sensor node is missing on an
  MY285 (or the firmware exposes a different path), the daemon **panics** inside the poll loop
  instead of degrading gracefully.
* Only the first byte of the 2-byte sysfs file is interpreted; the second byte's meaning is
  unverified (MiniUI parses the same file as a single integer).
* The hinge is polled, not interrupt-driven: worst-case latency ≈ 500 ms.
* Closing the hinge while the in-game menu is open does not do anything special — the menu owns
  the input while it is open.
* While the lid is closed, `self.keys[Key::LidClose]` stays `true` (it is only cleared by the
  `Released` event). The in-game menu can only be opened when **no key other than Menu** is
  held (`self.keys.iter().all(|(k, pressed)| k == Key::Menu || !pressed)` in
  `crates/alliumd/src/alliumd.rs`), so a closed lid blocks the menu for as long as it is closed.
* The `ffi` crate is unrelated to the lid: it only binds the audio header `mi_ao.h` for
  `myctl`. The hinge is pure sysfs.
