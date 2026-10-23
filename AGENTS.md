# AGENTS.md

Guidance for AI agents working in this repository.

## What this is

Allium is a custom launcher/UI for the **Miyoo Mini, Miyoo Mini Plus and Miyoo Mini Flip**
handhelds (Rust workspace, ~ARM Linux). It replaces the stock UI with a faster, cleaner
frontend, and ships RetroArch for playing games.

## Layout

|Path|Contents|
|---|---|
|`crates/common/`|shared library: `platform/` (hardware abstraction), `view/` (UI widgets), settings, SQLite DB|
|`crates/alliumd/`|the daemon: keys, battery, power/sleep policy, OSD, spawning apps|
|`crates/allium-launcher/`|main menu UI (Recents / Games / Apps / Settings)|
|`crates/allium-menu/`|in-game overlay menu (library + binary)|
|`crates/activity-tracker/`, `crates/screenshot-viewer/`|standalone apps|
|`crates/ffi/`, `crates/myctl/`|vendor C bindings (audio only) + tiny hardware CLI|
|`crates/say/`, `crates/show/`, `crates/screenshot/`|one-shot framebuffer helpers|
|`static/`|files deployed to the SD card (locales, themes, images, apps)|
|`docs/`|developer documentation — start at `docs/README.md`|

## Commands

```bash
make simulator bin=allium-launcher   # run the main menu on the desktop (no device needed)
make simulator bin=allium-menu
make build                            # cross-build for armv7-unknown-linux-gnueabihf
make all                              # full build + RetroArch into dist/
make deploy                           # deploy to $SDCARD_PATH
cargo test
cargo fmt && cargo clippy
```

## Key concepts

* **`alliumd` owns power.** The launcher/menus never touch hardware power state. Sleep,
  suspend, idle timeout and shutdown are all in `crates/alliumd/src/alliumd.rs`.
* **"Suspend"** = screen off + `SIGSTOP` of the foreground app (the device stays powered).
  **"Auto sleep"** = idle timeout, which actually *shuts the device down*.
* **The hinge (lid)** exists only on the Miyoo Mini Flip (MY285). It is a hall sensor read by
  `LidSwitchPoller` in `crates/common/src/platform/miyoo/evdev.rs` and surfaces as the synthetic
  key `Key::LidClose` (press = closed, release = opened). Closing it fires `lid_close_action`
  (default: shutdown); opening it wakes a suspended device.
* **The launcher `execve`s into the game** (on the device build), so the daemon's child handle *is*
  the game — that is what makes `SIGSTOP`/`SIGCONT` suspend work.
* **Settings are JSON files** under `<SD>/.allium/state/`. `alliumd` reads `power.json` once at
  startup, so power-setting changes need a restart (the UI shows a toast).
* **Three platform backends** behind the `Platform` trait: `miyoo`, `simulator`, `mock`.
  Use `DefaultPlatform`, never a concrete backend.

## Where to look

|Topic|File|
|---|---|
|Power / sleep / shutdown|`crates/alliumd/src/alliumd.rs`|
|Hinge (lid) detection|`crates/common/src/platform/miyoo/evdev.rs`|
|Screen blank / brightness|`crates/common/src/platform/miyoo/screen.rs`|
|Power settings model|`crates/common/src/power.rs`|
|Power settings UI|`crates/allium-launcher/src/view/settings/power.rs`|
|Paths, timeouts, thresholds|`crates/common/src/constants.rs`|
|Volume/brightness OSD|`crates/alliumd/src/osd.rs`|

Full details: [`docs/power-management.md`](docs/power-management.md),
[`docs/hinge-lid.md`](docs/hinge-lid.md), [`docs/architecture.md`](docs/architecture.md).

## Conventions

* `anyhow::Result` for errors, `log` for logging, `tokio` for async.
* Keep the daemon event loop in `alliumd.rs` responsive: never block the `select!` loop.
* New UI strings go into `static/.allium/locales/en-US/main.ftl` first, then translations.
* Settings are plain JSON with serde; there is no migration framework — a file that fails to
  parse is deleted and defaults are used.
