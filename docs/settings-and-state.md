# Settings & Persistent State

## Where everything lives

All Allium state is on the SD card under `.allium/`. The base directory is resolved in
`crates/common/src/constants.rs`:

```rust
pub static ref ALLIUM_BASE_DIR: PathBuf = PathBuf::from(
    &env::var("ALLIUM_BASE_DIR").map_or_else(|_| ALLIUM_SD_ROOT.join(".allium"), PathBuf::from)
);
pub static ref ALLIUM_SD_ROOT: LazyLock<PathBuf> = /* /mnt/SDCARD/ on device */;
```

`ALLIUM_BASE_DIR` / `ALLIUM_SD_ROOT` / `ALLIUM_GAMES_DIR` / `ALLIUM_APPS_DIR` /
`ALLIUM_DATABASE` can all be overridden with environment variables (this is how the simulator
and the tests redirect state into a scratch directory).

### State files

|Constant|Path (relative to `.allium/`)|Contents|
|---|---|---|
|`ALLIUMD_STATE`|`state/alliumd.json`|volume, brightness, RTC timestamp|
|`ALLIUM_LAUNCHER_STATE`|`state/allium-launcher.json`|launcher tab + list state|
|`ALLIUM_MENU_STATE`|`state/allium-menu.json`|in-game menu state|
|`ALLIUM_GAME_INFO`|`state/current_game`|`GameInfo` of the running game|
|`ALLIUM_POWER_SETTINGS`|`state/power.json`|`PowerSettings` (see below)|
|`ALLIUM_STYLESHEET`|`state/stylesheet.json`|theme / colours / fonts|
|`ALLIUM_DISPLAY_SETTINGS`|`state/display.json`|contrast, hue, colour temperature|
|`ALLIUM_LOCALE_SETTINGS`|`state/locale.json`|language|
|`ALLIUM_WIFI_SETTINGS`|`state/wifi.json`|WiFi configuration|
|`ALLIUM_UPDATE_SETTINGS`|`state/update.json`|OTA update state|
|`ALLIUM_TIMEZONE`|`state/timezone`|timezone name|
|`ALLIUM_THEME_STATE`|`state/theme`|selected theme|

Other locations:

|Constant|Path|Contents|
|---|---|---|
|`ALLIUM_DATABASE`|`<SD>/Saves/CurrentProfile/allium.db`|SQLite: games, guides, directories, game sessions|
|`ALLIUM_LOCALES_DIR`|`.allium/locales`|Fluent `.ftl` files, one dir per language|
|`ALLIUM_THEMES_DIR`|`<SD>/Themes`|themes|
|`ALLIUM_SCREENSHOTS_DIR`|`<SD>/Saves/CurrentProfile/screenshots`|save-state screenshots|

## The Power settings page

Data model — `crates/common/src/power.rs`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PowerSettings {
    pub power_button_action: PowerButtonAction,
    pub lid_close_action: PowerButtonAction,
    pub auto_sleep_when_charging: bool,
    pub auto_sleep_duration_minutes: i32,
    #[serde(default)]
    pub volume_on_startup: VolumeOnStartup,
}

#[derive(Debug, Copy, Clone, Serialize, Deserialize, FromRepr, Default)]
pub enum PowerButtonAction { #[default] Suspend, Shutdown, Nothing }

#[derive(Debug, Copy, Clone, Serialize, Deserialize, FromRepr, Default)]
pub enum VolumeOnStartup { #[default] Restore, Muted }
```

Defaults:

|Field|Default|
|---|---|
|`power_button_action`|`Suspend`|
|`lid_close_action`|`Shutdown`|
|`auto_sleep_when_charging`|`true` (auto sleep **allowed** while charging)|
|`auto_sleep_duration_minutes`|`5`|
|`volume_on_startup`|`Restore`|

Persistence:

```rust
pub fn load() -> Result<Self> {
    if ALLIUM_POWER_SETTINGS.exists() {
        let file = File::open(ALLIUM_POWER_SETTINGS.as_path())?;
        if let Ok(json) = serde_json::from_reader(file) { return Ok(json); }
        warn!("failed to read power file, removing");
        fs::remove_file(ALLIUM_POWER_SETTINGS.as_path())?;   // self-healing, no migrations
    }
    Ok(Self::new())
}
```

There is **no schema version and no migration mechanism** for settings. If the JSON cannot be
deserialised (e.g. a field was renamed), the file is deleted and defaults are used. The only
compatibility mechanism is `#[serde(default)]` on `volume_on_startup`.

## The UI

`crates/allium-launcher/src/view/settings/power.rs` renders a `SettingsList` with one row per
setting:

|Index|Label (locale key)|Widget|Range / options|
|---|---|---|---|
|0|`settings-power-auto-sleep-when-charging`|`Toggle`|on/off|
|1|`settings-power-auto-sleep-duration-minutes`|`Number`|0–60, step 5; `0` renders as `settings-power-auto-sleep-duration-disabled`|
|2|`settings-power-volume-on-startup`|`Select`|Restore / Muted|
|3|`settings-power-power-button-action`|`Select`|Suspend / Shutdown / Nothing|
|4|`settings-power-lid-close-action`|`Select`|Suspend / Shutdown / Nothing — **only if `has_lid()`**|

Changing a row emits `Command::ValueChanged(index, value)`, which the page maps back to a field
by index:

```rust
match i {
    0 => self.power_settings.auto_sleep_when_charging = val.as_bool().unwrap(),
    1 => self.power_settings.auto_sleep_duration_minutes = val.as_int().unwrap(),
    2 => self.power_settings.volume_on_startup = VolumeOnStartup::from_repr(val.as_int().unwrap() as usize).unwrap_or_default(),
    3 => self.power_settings.power_button_action = PowerButtonAction::from_repr(val.as_int().unwrap() as usize).unwrap_or_default(),
    4 => self.power_settings.lid_close_action = PowerButtonAction::from_repr(val.as_int().unwrap() as usize).unwrap_or_default(),
    _ => unreachable!("Invalid index"),
}
self.power_settings.save()?;
```

Then a toast is shown:

```rust
async fn toast_needs_restart_for_effect(res: &Resources, commands: &Sender<Command>) -> Result<()> {
    let message = res.get::<Locale>().t("settings-needs-restart-for-effect");
    Ok(commands.send(Command::Toast(message, Some(Duration::from_secs(5)))).await?)
}
```

The toast is required because **`alliumd` loads `PowerSettings` exactly once**, at startup
(`crates/alliumd/src/alliumd.rs`, `AlliumD::new`). Changing the power button or lid behaviour
therefore needs a device restart to take effect.

Note that the lid row is conditional, so the index mapping is positional: if a row is inserted,
both the `buttons` vector and the `match i` block must be updated.

## Settings container

`crates/allium-launcher/src/view/settings/mod.rs` lists the top-level settings pages:

|Index|Page|Condition|
|---|---|---|
|0|System Update|WiFi only|
|1|WiFi|WiFi only|
|2|Clock|always|
|3|**Power**|always|
|4|Display|always|
|5|Theme|always|
|6|Language|always|
|7|About|always|

When the device has no WiFi (`has_wifi() == false`, i.e. the Miyoo Mini MY283) the indices are
shifted by 2.

Per-page selection state (`ChildState { selected }`) is persisted inside the launcher state file
`state/allium-launcher.json`, and is only restored when the Settings tab was the last visible
tab.

## Localisation

Fluent (`.ftl`) files live in `static/.allium/locales/<lang>/main.ftl` and are installed to
`<SD>/.allium/locales/`. The loader uses `en-US` as the fallback locale
(`crates/common/src/locale.rs`), so a missing key in another language falls back to English.

Languages shipped: `en-US`, `en-GB`, `de-DE`, `es-ES`, `es-LA`, `fr-FR`, `id-ID`, `it-IT`,
`ja-JP`, `ro-RO`, `zh-CN`, `zh-HK`, `zh-TW`.

Power-related keys (from `en-US/main.ftl`):

```ftl
settings-power = Power
settings-power-power-button-action = Power Button Action
settings-power-power-button-action-suspend = Suspend
settings-power-power-button-action-shutdown = Shutdown
settings-power-power-button-action-nothing = Nothing
settings-power-lid-close-action = Lid Close Action
settings-power-auto-sleep-when-charging = Auto Sleep When Charging
settings-power-auto-sleep-duration-minutes = Auto Sleep Duration (Minutes)
settings-power-auto-sleep-duration-disabled = Disabled
settings-power-volume-on-startup = Volume on Startup
settings-power-volume-on-startup-restore = Restore
settings-power-volume-on-startup-muted = Muted
settings-needs-restart-for-effect = You need to restart the device for changes to take effect.
```

`en-GB` does not define the power strings and relies on the `en-US` fallback.

## Migrations

`migrations/` (repo root) and `static/.allium/migrations/` contain **device data** migrations
shipped to the SD card, each with a `name.txt` and a `run.sh`:

* `0000-retroarch-config`
* `0001-retroarch-core-overrides`
* `0002-move-screenshots`
* `0003-gpsp-sav-to-srm`

These are unrelated to the JSON settings above. Database schema changes are handled separately
by `rusqlite_migration` in `crates/common/src/database.rs`.
