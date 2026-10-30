# Using Mods

## Install the loader (once)
Put `ModLoader\`, `mods\` and `Launch Geoplanetical (Mods).bat` next to `Geoplanetical.exe`. Close the game, then start it with the `.bat`.
On the main menu a short text in the top-left shows how many mods loaded. You need Python 3 installed (`winget install Python.Python.3.12`); the `.bat` uses it to run `loader.py`.

If you prefer Steam's Play button, set the launch options to:
```
python "<game folder>\ModLoader\loader.py" %command%
```

## Install a mod
Drag its folder into `mods\` so you get `mods\<mod-id>\mod.json`, **or** drop a single locked `<mod-id>.geomod` file into `mods\`. Restart the game. To remove it, delete the folder (or just untick it).

## Turn mods on and off
Press **HOME** in the game, tick or untick, then press **Restart now**. Changes always apply on the next start, because mods patch the
game's code while it loads. The choices are stored in `mods\_loader.json`.

Folders whose name starts with `_` or `.` in `mods\` are ignored (handy for parking a mod).

## Command line
`python ModLoader\loader.py [launch|patch|restore|status] [--safe]`

| Command | Meaning |
|---|---|
| `launch` | patch if needed, then start the game (what the `.bat` does) |
| `--retry-native` | try the loader's key module again after it crashed once (see [Locking and Sealing](Locking-and-Sealing.md)) |
| `--safe` | start with **all mods off** (use this if a mod breaks the game) |
| `patch` | apply the loader hook without starting |
| `restore` | put the original game files back, no loader |
| `status` | show whether the hook is installed and healthy |

After a Steam update or "verify integrity", the hook is overwritten. Just launch with the `.bat` again and it re-applies.

## Where things are
| Thing | Location |
|---|---|
| Loader log | `ModLoader\loader.log` |
| Backups of the original files | `ModLoader\backup\` |
| Patched worker cache | `ModLoader\cache\` (safe to delete) |
| Your saves | `%APPDATA%\Geoplanetical\saves\*.gsav` |
| Gemini settings and logs | `%APPDATA%\Geoplanetical\gemini.json`, `gemini-chat.jsonl`, `gemini-decisions.jsonl` |

## Troubleshooting
* **A mod shows "failed" in the HOME panel**: the red text is the reason (bad `mod.json`, or a patch that no longer matches the game version).
  A failed mod applies *nothing*, so it can't half-break the game.
* **Game won't start / black screen**: run `python ModLoader\loader.py launch --safe`. If that works, a mod is the culprit; disable them one by one.
* **Everything stopped after a Steam update**: launch with the `.bat` again; if mods still fail, the game's code changed and those mods need new anchors
  (see [Patching the Game](Patching-the-Game.md)).
* **No mod toast at all, and `status` says the previous start crashed**: the key module guard switched mods off after a crash; run with `--retry-native`, check `ModLoader\loader.log`.
* **"package is damaged, edited, or was made for a different loader"**: the `.geomod` was modified or built with another loader secret; get a fresh copy.
* **Want a clean game**: `python ModLoader\loader.py restore`.
* **Console shows `ERR_FILE_NOT_FOUND`**: harmless; the loader now answers missing files quietly.
