# Architecture

Allium is a Rust workspace (`Cargo.toml`) of small binaries plus one shared library.

```
crates/
  common/          shared library: platform abstraction, UI widget toolkit, settings, DB
  alliumd/         the supervisor daemon (power, keys, battery, spawning, OSD)
  allium-launcher/ the main menu UI (Recents / Games / Apps / Settings tabs)
  allium-menu/     the in-game overlay menu (also a library, embedded in alliumd)
  activity-tracker/ standalone app: playtime & session history
  screenshot-viewer/ standalone app: browse saved screenshots
  ffi/             bindgen bindings to the Miyoo vendor C lib (mi_ao.h) — audio only
  myctl/           tiny CLI for hardware (volume, display blank); the only user of `ffi`
  say/             one-shot: draw a centred text message on the framebuffer
  screenshot/      one-shot: capture /dev/fb0 to a PNG (optionally with rumble)
  show/            one-shot: write an image / darken / clear the framebuffer
```

Build & packaging is driven by the `Makefile`; binaries land in `dist/`.

## Process model

```
        ┌────────────────────┐
        │      alliumd       │  daemon: keys, battery, power policy, OSD
        │  (tokio, unix)     │
        └─────────┬──────────┘
                  │ spawn / wait (tokio::process::Child)
                  │ signals: SIGSTOP, SIGCONT, SIGTERM, SIGKILL
                  ▼
        ┌────────────────────┐
        │  allium-launcher   │  ──execve──▶  RetroArch (the game)
        └────────────────────┘        (the launcher *becomes* the game)
```

Key consequence: `alliumd`'s child handle **is** the game process after a launch, because the
launcher replaces its own image with `execve` (`crates/allium-launcher/src/allium_launcher.rs`,
handling of `Command::Exec`; the `exec()` call is under `#[cfg(feature = "miyoo")]`, while the
simulator build execs `/bin/sh -c "make simulator bin=allium-menu"` instead). This is what makes
`SIGSTOP`/`SIGCONT` suspend work on a running game, and what makes the "is a game running?"
check (`ALLIUM_GAME_INFO` exists) meaningful.

## Channels between processes

There is **no** socket or D-Bus between the daemon and the UIs. Communication happens through:

1. **Process tree + POSIX signals** — `SIGSTOP`/`SIGCONT` (suspend), `SIGTERM`/`SIGKILL`
   (shutdown). See `signal()` / `terminate()` in `crates/alliumd/src/alliumd.rs`.
2. **RetroArch UDP network command interface** — `crates/common/src/retroarch.rs`,
   `127.0.0.1:55355` (`RETROARCH_UDP_SOCKET`). Used for `Pause`/`Unpause` around the in-game
   menu and `GetInfo` to query disk/state slots.
3. **Files on the SD card** — the real "shared database". All state lives under
   `<SD>/.allium/state/` (see [settings-and-state.md](settings-and-state.md)).
4. **One-shot helper binaries** — the daemon shells out to `say` ("charging", "powering-off"),
   `show` (`--darken` on shutdown), `screenshot` (Menu+Power), and the platform layer shells out
   to `myctl` for volume.
5. **In-process command bus** — inside each UI binary, `common::command::Command` is sent over a
   `tokio::sync::mpsc` channel and drained by the app's event loop
   (`crates/allium-launcher/src/allium_launcher.rs`).

## The in-game menu

`allium-menu` is built both as a library and a binary. `alliumd` links the library and runs it
on a dedicated OS thread with its own Tokio runtime handle (`MenuHandle` in
`crates/alliumd/src/alliumd.rs`):

* The **Menu key is owned by the daemon**, not by the launcher. The launcher explicitly ignores
  Menu key presses (`crates/allium-launcher/src/allium_launcher.rs`: "Ignore menu key presses",
  with the `Released(Menu)` event still forwarded) so that the menu can be opened over a running
  game.
* On Menu release (pressed alone, and only while a game is running): the daemon sends
  `RetroArchCommand::Pause`, hides the OSD, and hands a `RetroArchInfo` to the menu thread.
* When the menu closes, the daemon gets the "done" signal and sends `RetroArchCommand::Unpause`.

## Launcher state machine

`crates/allium-launcher/src/view/app.rs` holds `AppState`:

```rust
struct AppState {
    selected: usize,          // 0 Recents, 1 Games, 2 Apps, 3 Settings
    recents: RecentsState,
    games: GamesState,
    apps: AppsState,
    settings: SettingsState,
}
```

* Tab switching is cyclic (`next()`/`prev()` use `rem_euclid(4)`).
* `GamesState`/`AppsState` are `EntryListState<S>`; the nested `child` field is the
  directory-navigation stack.
* `RecentsState` is `#[serde(untagged)]` — carousel or list, chosen by the stylesheet.
* The state is persisted to `<SD>/.allium/state/allium-launcher.json`.

## Launching a game

1. User presses A on an entry → `EntryList::select_entry()`
   (`crates/allium-launcher/src/view/entry_list.rs`).
2. For a game: `ConsoleMapper::launch_game`
   (`crates/allium-launcher/src/consoles.rs`) resolves console + core, increments the play
   count, builds a `GameInfo` and writes it to `<SD>/.allium/state/current_game`.
3. The launcher sends `Command::Exec(game_info.command())`; the event loop saves the launcher
   state, clears the screen and `execve`s RetroArch.
4. `alliumd` sees the child exit → records play time into the SQLite DB, deletes
   `current_game`, and spawns the launcher again.

**Resume across reboots**: at daemon start, `spawn_main()` checks `GameInfo::load()`. If a game
info file exists, the game is relaunched automatically. Holding **Menu** or **B** during startup
deletes the game info instead ("bypass game resume").

**Apps** (`.pak` directories under `Apps/`) are launched the same way but without a `GameInfo`,
so no play time is recorded.

## Shared library (`common`)

* `platform/` — the `Platform` trait and backends (see [platform-layer.md](platform-layer.md)).
* `view/` — the widget toolkit (`View` trait, `SettingsList`, `Toggle`, `Number`, `Select`,
  `ButtonHints`, `Row`, toast, etc.).
* `command.rs` — the in-app command enum.
* `resources.rs` — a type-map (`Resources`) holding `Locale`, `Stylesheet`, `ConsoleMapper`, …
* `database.rs` — SQLite (`rusqlite`) with `games`, `guides`, `directories`, `game_sessions`,
  `key_value` tables; migrations via `rusqlite_migration`.
* `game_info.rs` — the `current_game` record.
* `retroarch.rs` — RetroArch UDP client.
* `power.rs` — `PowerSettings` (see [power-management.md](power-management.md)).
* `stylesheet.rs`, `locale.rs` — theming and Fluent-based localisation.
