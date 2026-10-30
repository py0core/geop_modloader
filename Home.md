# Geoplanetical Modding Wiki

Geoplanetical is an Electron app. The **Mod Loader** lets you change what the game does without ever editing the game's files:
mods are applied *in memory* every time the game starts, so Steam updates and "verify files" can't be broken by a mod, and
removing a mod removes its effect completely.

| Page | Read it if you want to... |
|---|---|
| [Using Mods](Using-Mods.md) | install mods, turn them on/off, fix problems |
| [Making a Mod](Making-a-Mod.md) | write your first mod; full `mod.json` reference |
| [Patching the Game](Patching-the-Game.md) | find the code you want to change and change it safely |
| [Main Process, Routes and UI](Main-Process-Routes-and-UI.md) | give a mod its own panel, files, network access |
| [Locking and Sealing](Locking-and-Sealing.md) | encrypt a mod, add menu settings, rebuild the sealed loader |
| [Game Internals](Game-Internals.md) | what we know about the game's code, data tables and saves |
| [Bundled Mods](Bundled-Mods.md) | what ships in the `mods` folder and how each is built |

## The 60-second picture

```
Geoplanetical\
  Geoplanetical.exe
  Launch Geoplanetical (Mods).bat   <- start the game with this
  ModLoader\                        <- the loader itself (loader.py, runtime\, wiki\)
  mods\                             <- one folder per mod, each with a mod.json
    save-editor\  game-tweaks\  slow-time\  gemini-*\
```

1. `loader.py` makes one small change to the game's Electron entry point (`electron/main.js`) so it loads `ModLoader/runtime/core.js` first.
   The original is backed up in `ModLoader\backup\`; `python ModLoader\loader.py restore` undoes it.
2. `core.js` reads every `mods/*/mod.json`, and intercepts the game's own file loads, handing the game **patched copies of its code**.
3. Mods can also run code in Electron's main process (`main`), inject scripts and CSS into the game window (`renderer`, `css`),
   and talk to each other through a tiny local API.

## Hotkeys

| Key | What |
|---|---|
| **HOME** | Mod panel (switch mods on/off). Configurable in `mods/_loader.json` (`hotkey`). |
| **F8** | Save Editor |
| **F9** | Game Tweaks |

> HOME is also the game's "camera home" key; the loader takes it over. Change `hotkey` in `mods/_loader.json` if you want it back.
