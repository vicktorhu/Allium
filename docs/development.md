# Development

## Requirements

* `make`, `cargo`, `zip`, `clang` (`libclang-dev`), `patchelf`
* [`cargo-zigbuild`](https://github.com/rust-cross/cargo-zigbuild) and `zig`
  (`pip install cargo-zigbuild`)
* On macOS: `./scripts/setup-mac.sh`

Rust toolchain is pinned in `rust-toolchain.toml`; the cross target is
`armv7-unknown-linux-gnueabihf` with glibc 2.28.

## Common commands

```bash
make all                      # build everything + RetroArch, copy into dist/
make build                    # cross-build the Rust binaries for the device
make debug                    # cross-build without --release
make simulator bin=allium-launcher   # run the main menu on the desktop
make simulator bin=allium-menu       # run the in-game menu on the desktop
make deploy                   # copy the update files to $SDCARD_PATH
make deploy-all               # copy everything to a fresh SD card
```

The simulator target needs `make`, `cargo`, and a graphical environment (winit + softbuffer).
It runs with `ALLIUM_DATABASE`, `ALLIUM_BASE_DIR` and `ALLIUM_SD_ROOT` pointed at
`simulator/` in the repo.

To deploy to a device, create `local.mk` (git-ignored):

```makefile
SDCARD_PATH=/path/to/sdcard
```

or export `SDCARD_PATH` in the environment.

## Cargo features

|Feature|Effect|
|---|---|
|`miyoo`|Real hardware backend (`evdev`, `framebuffer`, `sysfs_gpio`)|
|`simulator`|Desktop backend (`winit`, `softbuffer`)|
|`console`|`alliumd` only: starts a `tokio console` on port 6669|

## Repository layout

```text
crates/        the Rust workspace
static/        files deployed to the SD card (locales, themes, images, apps, updater)
dist/          build output (generated)
simulator/     scratch dir used by `make simulator`
migrations/    device data migrations shipped to the SD card
third-party/   vendored RetroArch patch, dufs, collie, my283 toolchain headers/libs
scripts/       helper scripts
```

## Conventions

* **Formatting**: `cargo fmt` (rustfmt defaults, 4-space indent).
* **Linting**: `cargo clippy`; the binaries use `#![deny(clippy::all)]`.
* **Errors**: `anyhow::Result` everywhere in application code.
* **Logging**: `log` + `simple_logger`; `info!` for lifecycle events, `debug!`/`trace!` for
  detail, `warn!`/`error!` for problems.
* **Async**: `tokio`; the platform trait is `#[async_trait(?Send)]` because it is not `Send`.
* **Settings**: JSON via `serde`, one file per concern under `.allium/state/`.
* **UI**: the `View` trait in `crates/common/src/view/`; widgets draw into a `Display` and
  report whether they redrew. Commands are pushed through `common::command::Command`.
* **Localisation**: Fluent `.ftl` files; add the key to `en-US` first, then translate.

## Testing

```bash
cargo test
```

Unit tests live next to the code they test (for example `test_parse_firmware` in
`crates/common/src/platform/miyoo/mod.rs`). The mock platform (`crates/common/src/platform/mock.rs`)
is used when neither `miyoo` nor `simulator` is enabled, so host builds compile and test
without hardware.

## Deploying / updating

* `dist/.allium/bin/` — `alliumd`, `allium-launcher`
* `dist/.tmp_update/bin/` — `screenshot`, `say`, `show`, `myctl`
* `dist/Apps/Activity Tracker.pak/`, `dist/Apps/Screenshot Viewer.pak/` — the standalone apps

The device's updater (`static/.tmp_update/updater`) starts the daemon:

```sh
"$ROOT"/.allium/bin/alliumd >"$ROOT"/allium.log 2>&1
```

so `allium.log` on the SD card is the first place to look when debugging power behaviour on a
real device.
