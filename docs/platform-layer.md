# Platform Layer

All hardware access is funnelled through one trait, `common::platform::Platform`
(`crates/common/src/platform/mod.rs`), with three backends selected by cargo feature:

|Feature|Backend|Notes|
|---|---|---|
|`miyoo`|`miyoo::MiyooPlatform`|Real device|
|`simulator`|`simulator::SimulatorPlatform`|Desktop winit window + softbuffer|
|*(neither)*|`mock::MockPlatform`|No-op, for host builds/tests|

Downstream crates never name a concrete backend; they use the alias `DefaultPlatform`
(`crates/common/src/platform/mod.rs`):

```rust
pub type DefaultPlatform = miyoo::MiyooPlatform;      // feature = "miyoo"
pub type DefaultPlatform = simulator::SimulatorPlatform; // feature = "simulator"
pub type DefaultPlatform = mock::MockPlatform;           // default build
```

## The trait

```rust
#[async_trait(?Send)]           // Platform is not threadsafe because it is ?Send
pub trait Platform {
    type Display: Display;
    type Battery: Battery + 'static;
    type SuspendContext;        // opaque, carries state across suspend

    fn new() -> Result<Self> where Self: Sized;
    fn display(&mut self) -> Result<Self::Display>;
    fn display_partial(&mut self) -> Result<Self::Display> { self.display() }
    fn battery(&self) -> Result<Self::Battery>;
    async fn poll(&mut self) -> KeyEvent;
    fn shutdown(&self) -> Result<()>;
    fn suspend(&self) -> Result<Self::SuspendContext>;
    fn unsuspend(&self, ctx: Self::SuspendContext) -> Result<()>;
    fn set_volume(&mut self, volume: i32) -> Result<()>;
    fn get_brightness(&self) -> Result<u8>;
    fn set_brightness(&mut self, brightness: u8) -> Result<()>;
    fn set_display_settings(&mut self, settings: &mut DisplaySettings) -> Result<()>;
    fn device_model() -> String;
    fn firmware() -> String;
    fn has_wifi() -> bool;
    fn has_lid() -> bool;       // ← hinge support
    fn daemon(&self) {}         // optional per-platform background tasks
}
```

`SuspendContext` is an associated type so each platform decides what must survive a suspend:

* `miyoo::SuspendContext { brightness: u8 }` — the brightness is saved before the screen is
  blanked and restored on wake.
* `()` for simulator and mock.

Because the context is `()` on desktop, call sites carry `#[allow(clippy::let_unit_value)]`.

## Keys

```rust
pub enum KeyEvent { Pressed(Key), Released(Key), Autorepeat(Key) }

pub enum Key {
    Up, Down, Left, Right, A, B, X, Y, Start, Select, L, R,
    Menu, L2, R2, Power, VolDown, VolUp, LidClose, Unknown,
}
```

`Key::LidClose` is a **synthetic** key: it is not an evdev key, it is produced by the hinge
sensor poller. See [hinge-lid.md](hinge-lid.md).

## Miyoo backend (`crates/common/src/platform/miyoo/`)

|File|Responsibility|
|---|---|
|`mod.rs`|`MiyooPlatform`, device-model detection, suspend/unsuspend, display settings, shutdown|
|`evdev.rs`|`/dev/input/event0` key events + hinge (hall sensor) polling|
|`screen.rs`|brightness (PWM) and framebuffer blanking|
|`framebuffer.rs`|`/dev/fb0` display, frame read/write, the "stamp"/`RectHold` mechanism|
|`volume.rs`|volume curve, shells out to `myctl volume`, persists raw value in `/tmp/volume`|
|`battery/`|per-model battery backends (`miyoo283.rs` ADC via `/dev/sar`, `miyoo354.rs` via `/customer/app/axp_test`)|
|`usb_audio.rs`|MY285-only USB-audio routing (GPIO45 → GPIO44)|

### Device model detection

```rust
fn detect_model() -> MiyooDeviceModel {
    if Path::new("/dev/input/event1").exists()      { Miyoo285 }  // Mini Flip
    else if Path::new("/customer/app/axp_test").exists() { Miyoo354 } // Mini+
    else                                              { Miyoo283 }  // Mini
}
```

|Model|Name|WiFi|Hinge (lid)|
|---|---|---|---|
|MY283|Miyoo Mini|no|**no**|
|MY285|Miyoo Mini Flip|yes|**yes**|
|MY354|Miyoo Mini+|yes|**no**|

Firmware version comes from `/etc/fw_printenv`, parsing `miyoo_version=`.

### Suspend / unsuspend on hardware

```rust
fn suspend(&self) -> Result<Self::SuspendContext> {
    let brightness = screen::get_brightness()?;
    let ctx = SuspendContext { brightness };
    screen::set_brightness(0)?;
    screen::blank(true)?;
    Ok(ctx)
}

fn unsuspend(&self, ctx: Self::SuspendContext) -> Result<()> {
    screen::blank(false)?;
    screen::set_brightness(ctx.brightness)?;
    Ok(())
}
```

Important: **there is no real device power management here.** "Suspend" only turns the screen off
(PWM brightness → 0, plus `GUI_SHOW 0 off` on `/proc/mi_modules/fb/mi_fb0`). The device stays
powered; freezing the running app is done by the daemon with `SIGSTOP`.

`screen.rs` details:

* brightness: `/sys/devices/soc0/soc/1f003400.pwm/pwm/pwmchip0/pwm0/duty_cycle`
  (`set_brightness` clamps to a minimum of `3`).
* blank: writes `GUI_SHOW 0 off` / `GUI_SHOW 0 on` to `/proc/mi_modules/fb/mi_fb0`.

### Shutdown

```rust
fn shutdown(&self) -> Result<()> {
    Command::new("sync").spawn()?.wait()?;
    match self.model {
        Miyoo283            => { Command::new("reboot").exec(); }
        Miyoo285 | Miyoo354 => { Command::new("poweroff").exec(); }
    }
}
```

`exec()` replaces the process, so the daemon never returns from `shutdown()`.

### Display settings

`set_display_settings` writes `csc ...` and `colortemp ...` lines to
`/proc/mi_modules/mi_disp/mi_disp0`, clamping contrast to ≥ 10 and each RGB channel to ≥ 15.
A failure to open the file is logged and ignored.

## Simulator backend (`crates/common/src/platform/simulator.rs`)

* winit window + softbuffer surface; size from `WIDTH` (752), `HEIGHT` (560), `SCALE` (1).
* `poll()` is non-blocking (`try_recv`), pumping window events on demand so the tokio runtime
  can make progress.
* `suspend()`/`unsuspend()` are **no-ops**, `SuspendContext = ()`.
* `has_lid()` returns `true` (so the hinge setting is visible in the simulator UI), but the
  simulator never emits `Key::LidClose` events — hinge behaviour cannot be exercised there.
* `SIMULATOR_CHARGING=1` makes the fake battery report "charging".

## Mock backend (`crates/common/src/platform/mock.rs`)

Every method is a no-op; `poll()` awaits forever. Used for host builds and `cargo test`.

## About the `ffi` crate

`crates/ffi` is **not** part of the suspend or GPIO path. It is a bindgen wrapper over the
Sigmastar audio header `third-party/my283/usr/include/mi_ao.h` and is used only by
`crates/myctl` (`crates/myctl/src/volume.rs`) to set the amplifier volume.

GPIO access elsewhere uses the `sysfs_gpio` crate (`usb_audio.rs`, `battery/miyoo283.rs`) or raw
writes to `/sys/class/gpio/gpio86/value` (`battery/miyoo354.rs`, the charging LED).
