# Power Management: Sleep, Suspend, Shutdown

Everything in this document is implemented in **`crates/alliumd/src/alliumd.rs`**, the supervisor
daemon. The daemon is the only component that owns power policy; the launcher and menus never
touch the hardware power state.

## Vocabulary (important)

|Term in Allium|What it actually does|
|---|---|
|**Suspend**|Screen off (brightness → 0, framebuffer blanked) **and** the foreground app is frozen with `SIGSTOP`. The device stays powered. Woken by releasing the power button or opening the hinge.|
|**Auto sleep**|An idle timer in the main event loop. Despite the name, this **shuts the device down** — it does *not* suspend.|
|**Charging suspend**|A one-shot flow entered at daemon start when the battery is charging: freeze the app, show "charging", suspend the screen, and power off if charging stops.|
|**Shutdown**|Save state, terminate the foreground app, darken the screen, "powering off" message, then `reboot`/`poweroff`.|

## The daemon event loop

`AlliumD::run_event_loop()` is a single `tokio::select!` loop. The arms, in order:

|Arm|Effect|
|---|---|
|`key_event = self.platform.poll()`|→ `handle_key_event`|
|`_ = sleep_until_wake(self.osd.next_wake())`|OSD (volume/brightness plate) tick|
|`_ = self.menu.done_rx.recv()`|in-game menu closed → `RetroArchCommand::Unpause`|
|`_ = tokio::time::sleep(auto_sleep_duration)`|**idle timeout → `handle_quit()` (shutdown)**|
|`_ = self.main.wait()`|the foreground app exited → record play time, respawn the launcher|
|`_ = sigint.recv()` / `sigterm.recv()`|→ `handle_quit()`|

Because the `sleep(auto_sleep_duration)` arm is *recreated* on every loop iteration, **any event
that wakes the loop resets the idle timer**. In practice, "idle" means "no key input" (plus the
OSD's own ~1 s tick while an overlay is visible).

## Auto sleep (idle timeout)

```rust
let auto_sleep_duration = match self.power_settings.auto_sleep_duration_minutes {
    0 => std::time::Duration::MAX,   // disabled
    t  => std::time::Duration::new(t as u64 * 60, 0),
};
...
_ = tokio::time::sleep(auto_sleep_duration) => {
    if !self.power_settings.auto_sleep_when_charging && battery.charging() {
        info!("battery charging, don't auto sleep");
    } else if Path::new("/tmp/stay_awake").exists() {
        info!("/tmp/stay_awake exists, don't auto sleep");
    } else {
        info!("idle timeout, shutting down");
        self.handle_quit().await?;
    }
}
```

Rules:

* `auto_sleep_duration_minutes` — 0 disables auto sleep entirely; otherwise the idle time in
  minutes (UI range 0–60, step 5). Default **5**.
* `auto_sleep_when_charging` — the naming is inverted relative to intuition:
  `true` (**default**) means *auto sleep is allowed while charging*; `false` means charging
  suppresses auto sleep.
* `/tmp/stay_awake` — a sentinel file that suppresses auto sleep. Created by the OTA update flow
  (`crates/allium-launcher/src/view/settings/system.rs`) so a download is never interrupted.
* Result of firing: **shutdown**, not suspend.

## Suspend

```rust
async fn handle_suspend(&mut self) -> Result<()> {
    info!("suspending...");
    self.hide_osd();
    let ctx = self.platform.suspend()?;          // screen off, brightness saved
    signal(&self.main, Signal::SIGSTOP)?;         // freeze the game / launcher

    loop {
        tokio::select! {
            key_event = self.platform.poll() => {
                if matches!(key_event, KeyEvent::Released(Key::Power))
                    || matches!(key_event, KeyEvent::Released(Key::LidClose)) {
                    self.keys[Key::Power] = false;
                    self.keys[Key::LidClose] = false;
                    break;                        // wake up
                }
            }
            _ = tokio::time::sleep(IDLE_TIMEOUT) => {   // 5 minutes
                info!("idle timeout, shutting down");
                signal(&self.main, Signal::SIGCONT)?;
                self.platform.unsuspend(ctx)?;
                self.handle_quit().await?;
                return Ok(());
            }
        }
    }

    info!("waking up from suspend...");
    signal(&self.main, Signal::SIGCONT)?;         // thaw the game
    self.platform.unsuspend(ctx)                  // screen back on
}
```

While suspended, the daemon is blocked inside `handle_suspend`, so the other `select!` arms
(OSD tick, menu-done, `main.wait()`, signal handling) are **not** running, and any key event
other than power/lid is swallowed without updating `self.keys`.

Sequence:

1. Hide the OSD overlay (so it does not flash on wake).
2. `Platform::suspend()` → brightness saved into `SuspendContext`, brightness set to 0,
   framebuffer blanked.
3. `SIGSTOP` the foreground child. Because the launcher `execve`s into RetroArch, this child
   *is* the game — the game is frozen exactly where it was, no save state is touched.
4. Wait for a wake event: **release** of the power button, or **release** of `Key::LidClose`
   (i.e. opening the hinge).
5. On wake: `SIGCONT` then `unsuspend()` (un-blank the screen, restore the previous brightness).
6. If nothing happens for `IDLE_TIMEOUT` (**5 minutes**, `constants::IDLE_TIMEOUT`), the device
   unsuspends and **shuts down** instead of staying suspended forever.

Note that the wake condition is the **release** of the power button. The release that *caused*
the suspend has already been consumed by `handle_key_event`, so waking requires a **new
press-and-release** of the power button (or opening the hinge).

## Power button

Handled in `handle_key_event` (the branch used when **Menu is not held**):

|Event|Action|
|---|---|
|`KeyEvent::Released(Key::Power)`|`power_button_action`: `Suspend` → `handle_suspend()`, `Shutdown` → `handle_quit()`, `Nothing` → ignored|
|`KeyEvent::Autorepeat(Key::Power)`|immediate `handle_quit()` (long-press = shutdown, regardless of the setting)|
|`KeyEvent::Released(Key::Power)` **with Menu held**|take a screenshot (`screenshot --rumble`) into `<SD>/Screenshots/`|

## Hinge (lid) close

Handled in the same branch, on `KeyEvent::Pressed(Key::LidClose)`:

|`lid_close_action`|Result|
|---|---|
|`Suspend`|`handle_suspend()` — screen off, app frozen; **opening the hinge wakes the device**|
|`Shutdown`|`handle_quit()` — full shutdown (this is the **default**)|
|`Nothing`|nothing happens on close — but note that *opening* the hinge still wakes a suspended device|

Detection details: [hinge-lid.md](hinge-lid.md).

## Charging suspend

Entered at daemon start, before the event loop, when the battery reports charging:

```rust
let mut battery = self.platform.battery()?;
battery.update()?;
if battery.charging() {
    self.handle_charging().await?;
}
```

`handle_charging()`:

1. Hide the OSD.
2. `SIGSTOP` the foreground app.
3. `say "<locale: charging>"` — draw the message on screen.
4. Wait 5 seconds, then `show -c` to clear the screen.
5. `platform.suspend()` — screen off.
6. Loop: poll the battery every second; if charging stopped → `platform.shutdown()`.
   A power-button release breaks the loop.
7. `SIGCONT` the app and `unsuspend(ctx)`.

## Shutdown

```rust
async fn handle_quit(&mut self) -> Result<()> {
    if self.is_terminating { return Ok(()); }
    self.hide_osd();
    self.state.time = Utc::now();
    self.state.save()?;                 // <SD>/.allium/state/alliumd.json (RTC, volume, brightness)

    if self.is_ingame() { self.update_play_time()?; }   // DB write

    terminate(&mut self.main).await?;   // SIGTERM, then SIGKILL after 5 s
    self.is_terminating = true;

    Command::new("show").arg("--darken").spawn()?.wait().await?;
    Command::new("say").arg(self.locale.t("powering-off")).spawn()?.wait().await?;
    tokio::time::sleep(Duration::from_millis(250)).await;

    self.platform.shutdown()?;          // reboot (MY283) / poweroff (MY285, MY354)
    Ok(())
}
```

`terminate()` sends `SIGTERM` and escalates to `SIGKILL` if the child has not exited within 5 s.

`AlliumDState::load()` also handles a broken RTC: if the stored time is in the future, the
daemon sets the system clock (`date --utc --set`) and syncs the hardware clock
(`/sbin/hwclock --systohc`), which is what makes "resume the game after a power cut" work.

Shutdown is also triggered by:

* `SIGINT` / `SIGTERM` sent to the daemon.
* Battery below `BATTERY_SHUTDOWN_THRESHOLD` (5 %) while not charging.
* The idle timeout (auto sleep), as described above.

## Battery monitoring

* The battery is refreshed every `BATTERY_UPDATE_INTERVAL` (10 s) at the top of the event loop.
* Below `BATTERY_WARNING_THRESHOLD` (15 %) and not charging → the charging LED blinks
  (0.5 Hz) via a spawned tokio task; the task is aborted when the battery recovers.
* Below `BATTERY_SHUTDOWN_THRESHOLD` (5 %) and not charging → `handle_quit()`.

## The OSD overlay

`crates/alliumd/src/osd.rs` draws the volume/brightness plate directly onto the framebuffer
while a game owns it. It has a 1 s hide timeout (`HIDE_TIMEOUT`). It is hidden explicitly
before suspend, charging and shutdown, because a leftover overlay would otherwise flash on
wake.

`foreground_repaints()` compares `/proc/<pid>/exe` of the child with the launcher path to decide
whether the app underneath repaints the framebuffer (and therefore whether the OSD needs a
"stamper" thread, `Refresh::Continuous`) or can be refreshed periodically
(`Refresh::Periodic`).

## Settings that drive all of this

`crates/common/src/power.rs`:

```rust
pub struct PowerSettings {
    pub power_button_action: PowerButtonAction,      // default Suspend
    pub lid_close_action: PowerButtonAction,          // default Shutdown
    pub auto_sleep_when_charging: bool,               // default true (i.e. allowed)
    pub auto_sleep_duration_minutes: i32,             // default 5
    #[serde(default)]
    pub volume_on_startup: VolumeOnStartup,           // default Restore
}

pub enum PowerButtonAction { Suspend, Shutdown, Nothing }
```

Loaded **once** at daemon start (`PowerSettings::load()?`), which is why every change in the
launcher shows the "you need to restart the device" toast. See
[settings-and-state.md](settings-and-state.md).

## Gotchas worth knowing before you change this code

* `PowerButtonAction` is converted through `FromRepr` with a bare `as usize` cast in the UI
  (`crates/allium-launcher/src/view/settings/power.rs`). Reordering the enum variants silently
  changes the meaning of the stored settings.
* `handle_suspend` wakes on `Released(Power)`/`Released(LidClose)`, but the *action* is bound to
  `Pressed(LidClose)` / `Released(Power)` — the asymmetry is deliberate (a lid is not a button),
  but it means "lid close = suspend" only takes effect on the closing edge.
* `self.keys[Key::Power]` / `self.keys[Key::LidClose]` are cleared on wake so that the key
  state does not stay latched.
* Auto sleep **shuts down**; if you want a real idle-suspend, that behaviour does not exist yet.
* The "disabled" case uses `std::time::Duration::MAX` as the sleep duration. This is safe with
  modern tokio: `tokio::time::sleep` computes `Instant::now().checked_add(duration)` and falls
  back to `Instant::far_future()` on overflow (verified against tokio 1.48/1.52 sources; the
  workspace resolves to 1.52.3), so no panic occurs.
* `handle_suspend` and `handle_charging` are `#[cfg(unix)]` only; the non-unix build of the
  daemon runs a reduced event loop with only the key-event and OSD arms.
