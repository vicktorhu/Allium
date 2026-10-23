# Allium Documentation

Allium is a custom launcher/UI for the Miyoo Mini, Miyoo Mini Plus and Miyoo Mini Flip
handhelds. This folder documents the codebase for developers and AI agents.

## Reading order

|Document|What it covers|
|---|---|
|[architecture.md](architecture.md)|The crates, how they talk to each other, process model, game launch/resume flow|
|[platform-layer.md](platform-layer.md)|The `Platform` trait and its three backends (Miyoo / Simulator / Mock), hardware access|
|[power-management.md](power-management.md)|**Sleep, suspend, auto-sleep, charging suspend and shutdown** — the full daemon power state machine|
|[hinge-lid.md](hinge-lid.md)|**Hinge (lid) open/close detection** on the Miyoo Mini Flip, and how lid events drive power behaviour|
|[settings-and-state.md](settings-and-state.md)|Where settings/state live on the SD card, the Power settings page, persistence rules|
|[development.md](development.md)|Build, simulator, deploy, tooling, conventions|

## TL;DR of the two topics you probably care about

**Sleep** is split into three distinct behaviours, all implemented in `crates/alliumd/src/alliumd.rs`:

1. *Suspend* (`handle_suspend`) — screen off + `SIGSTOP` of the foreground app. Woken by
   releasing the power button or opening the hinge. After 5 minutes suspended without a wake,
   the device shuts down. Configurable via `power_button_action` / `lid_close_action`.
2. *Auto sleep* (idle timeout in the main `select!` loop) — despite the name, this **shuts the
   device down**, it does not suspend. Configurable via `auto_sleep_duration_minutes`
   (`0` = disabled) and `auto_sleep_when_charging`.
3. *Charging suspend* (`handle_charging`) — entered at daemon start when the battery is
   charging; freezes the app, shows a "charging" message and suspends until charging stops.

**Hinge open/close** is implemented in `crates/common/src/platform/miyoo/evdev.rs`:
`LidSwitchPoller` reads the hall sensor at
`/sys/devices/soc0/soc/soc:hall-mh248/hallvalue` and converts edge transitions into synthetic
`Key::LidClose` press/release events. It only exists on the Miyoo Mini Flip (MY285), where
`Platform::has_lid()` returns `true`.

## Quick links

- Daemon: `crates/alliumd/src/alliumd.rs`
- Platform trait: `crates/common/src/platform/mod.rs`
- Miyoo platform: `crates/common/src/platform/miyoo/{mod.rs,evdev.rs,screen.rs}`
- Power settings model: `crates/common/src/power.rs`
- Power settings UI: `crates/allium-launcher/src/view/settings/power.rs`
- Constants (paths, timeouts): `crates/common/src/constants.rs`
- OSD overlay: `crates/alliumd/src/osd.rs`
