# AltLauncher Config Reference

This document covers how to write `AltLauncher.ini` config files for specific games, and how to contribute templates to the [AltLauncher-Templates](https://github.com/AetherCollective/AltLauncher-Templates) repository.

---

## Overview

Each game that uses AltLauncher needs a config file named `AltLauncher.ini` placed in the same directory as `AltLauncher.exe`. This file tells AltLauncher:

- What the game is called and how to launch it
- Where the game stores its save files (directories, files, and/or registry keys)
- Any timing or behavior overrides specific to the game

Environment variables set during AltLauncher's setup wizard serve as global defaults. The ini file can override any of them for a specific game.

---

## File Structure

An `AltLauncher.ini` has up to six sections:

```ini
[General]
[Settings]
[Profiles]
[Registry]
[Directories]
[Files]
```

`[Registry]`, `[Directories]`, and `[Files]` are all optional — include only the ones relevant to the game.

---

## [General]

Defines the game identity and launch parameters.

| Key | Required | Description |
|---|---|---|
| `Name` | Yes | Display name of the game. Used in window titles and profile folder names. |
| `Executable` | Yes | The game's executable filename (e.g. `hollow_knight.exe`). Do not include the full path — use `Path` for that. |
| `Path` | No | Working directory for launching the game. Defaults to the directory AltLauncher is in. |
| `LaunchFlags` | No | Optional command-line arguments passed to the game executable. |

### Example

```ini
[General]
Name=Hollow Knight
Executable=hollow_knight.exe
Path=C:\Program Files (x86)\Steam\steamapps\common\Hollow Knight
```

---

## [Settings]

Fine-tunes timing and behavior for the game. All keys are optional and fall back to defaults or environment variable values if omitted.

| Key | Description | Default |
|---|---|---|
| `MinWait` | Minimum time in seconds the process must run before AltLauncher considers it "properly launched". If the process exits faster than this, AltLauncher waits for it to reappear. | `0` |
| `MaxWait` | Maximum time in seconds to wait for the process to reappear after a fast exit. Pairs with `MinWait`. | `0` |
| `SaveDelay` | Extra time in milliseconds to wait after the game closes before saving. Useful for games that write saves asynchronously after the process exits. | `0` |
| `SafeMode` | Overrides the global `AltLauncher_SafeMode` environment variable for this game only. | Env var value |
| `SwitchMode` | Overrides the global `AltLauncher_SwitchMode` environment variable for this game only. | Env var value |

### MinWait / MaxWait explained

Some games launch through a stub process that immediately exits and hands off to a second process. Without `MinWait`/`MaxWait`, AltLauncher would incorrectly detect the game as already closed.

If the process dies in less than `MinWait` seconds, AltLauncher assumes it was a stub and waits up to `MaxWait` seconds for the real process to appear, then waits for that to finish.

```ini
[Settings]
MinWait=5
MaxWait=30
SaveDelay=2000
```

---

## [Profiles]

Overrides the global profile storage location for this game. Both keys are optional.

| Key | Description |
|---|---|
| `Path` | Root folder for profiles. Overrides `AltLauncher_Path`. |
| `SubPath` | Optional subfolder inside each profile. Overrides `AltLauncher_SubPath`. |

```ini
[Profiles]
Path=D:\GameSaves
SubPath=RPGs
```

---

## [Directories]

Defines folders to swap per-profile. Each entry is a key-value pair where:

- The **key** is a label used as the folder name inside the profile
- The **value** is the full path to the directory on disk

Windows environment variables are expanded automatically (e.g. `%USERPROFILE%`, `%APPDATA%`). The platform ID variables set by AltLauncher's setup wizard (`%SteamID3%`, `%SteamID64%`, `%UbisoftID%`, `%RockstarID%`) are also available.

```ini
[Directories]
Saves=%USERPROFILE%\AppData\Roaming\Hollow Knight\Saves
Screenshots=%USERPROFILE%\AppData\Roaming\Hollow Knight\Screenshots
```

### How directory swapping works

**On launch (Backup):**
1. The live directory is moved to `<dir>.AltLauncher-Backup`
2. The profile's copy of the directory is copied into place

**On close (Restore):**
1. The live directory's contents are copied back into the profile
2. Stale files (present in profile but not in live dir) are handled per Safe Mode
3. The live directory is removed
4. The backup is moved back into place

---

## [Files]

Defines individual files to swap per-profile. Same key-value format as `[Directories]`.

```ini
[Files]
Config=%APPDATA%\MyGame\config.ini
KeyBinds=%APPDATA%\MyGame\keybinds.cfg
```

### How file swapping works

**On launch (Backup):**
1. The live file is moved to `<file>.AltLauncher-Backup`
2. The profile's copy is placed at the live path

**On close (Restore):**
1. The live file is moved back into the profile
2. The backup is restored to the live path

If the file didn't exist when the game launched (i.e. the game created it fresh), Safe Mode determines what happens to it in the profile on restore.

---

## [Registry]

Defines registry keys to swap per-profile. Same key-value format, where the value is the full registry path.

```ini
[Registry]
GameSettings=HKCU\Software\MyGame\Settings
```

### How registry swapping works

**On launch (Backup):**
1. The live key is copied to `<key>.AltLauncher-Backup`
2. The live key is deleted
3. The profile's `.reg` file is imported

**On close (Restore):**
1. The live key is exported to the profile as a `.reg` file
2. The live key is deleted
3. The backup key is restored

---

## Safe Mode Reference

Safe Mode controls how stale files are handled during restore — files that exist in the profile but were not present in the live game directory when the game closed.

| Value | Behavior |
|---|---|
| `True` | Stale files are sent to the Recycle Bin |
| `False` | Stale files are permanently deleted |
| Unset | The entire directory is moved wholesale — no cleanup occurs |

Can be set globally via `AltLauncher_SafeMode` or per-game via `SafeMode` in `[Settings]`.

---

## Environment Variable Expansion

All path values in `[Directories]`, `[Files]`, and `[Registry]` support Windows environment variable expansion. Variables must be wrapped in `%` signs.

**Standard Windows variables:**

```
%USERPROFILE%   -> C:\Users\YourName
%APPDATA%       -> C:\Users\YourName\AppData\Roaming
%LOCALAPPDATA%  -> C:\Users\YourName\AppData\Local
%PROGRAMFILES%  -> C:\Program Files
```

**AltLauncher platform ID variables** (set during setup):

```
%SteamID3%      -> Steam3 ID (folder under userdata)
%SteamID64%     -> Steam64 ID
%UbisoftID%     -> Ubisoft Connect save folder ID
%RockstarID%    -> Rockstar Social Club ID
```

### Steam save path example

Steam cloud saves typically live at:

```
C:\Program Files (x86)\Steam\userdata\<SteamID3>\<AppID>\remote\
```

In a config:

```ini
[Directories]
Saves=C:\Program Files (x86)\Steam\userdata\%SteamID3%\480\remote
```

---

## Full Example

```ini
[General]
Name=Hollow Knight
Executable=hollow_knight.exe
Path=C:\Program Files (x86)\Steam\steamapps\common\Hollow Knight

[Settings]
SaveDelay=1000

[Directories]
Saves=%APPDATA%\..\LocalLow\Team Cherry\Hollow Knight
```

---

## Contributing a Template

Game-specific templates are maintained at [AetherCollective/AltLauncher-Templates](https://github.com/AetherCollective/AltLauncher-Templates).

To contribute a template for a new game:

1. Fork the templates repository
2. Create a folder named after the game (matching the `Name` key in the ini)
3. Add an `AltLauncher.ini` using the format described in this document
4. Test it — make sure save swapping works correctly for at least one full launch/close cycle
5. Open a pull request

When naming the folder and the `Name` key, use the game's exact title as it appears on Steam where possible, as AltLauncher uses the Steam App ID to match games to templates automatically.

---

## Tips for Finding Save Locations

- **PCGamingWiki** ([pcgamingwiki.com](https://www.pcgamingwiki.com)) is the most reliable source for save file locations across all platforms
- Use **Process Monitor** (from Sysinternals) filtered to the game's process to observe exactly which files and registry keys it reads and writes at runtime
- Steam cloud save paths almost always follow the `userdata\<SteamID3>\<AppID>\remote` pattern
- Check both `%APPDATA%` (Roaming) and `%LOCALAPPDATA%` — games use both
- Some games write to `%USERPROFILE%\Documents\My Games\` or `%USERPROFILE%\Saved Games\`
- Registry saves are less common but do appear — filter Process Monitor to `RegSetValue` operations to find them
