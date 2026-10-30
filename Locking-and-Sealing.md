# Locking Mods and the Sealed Loader

Two protections work together:

1. **Locked mods**: a mod folder can be packed into one encrypted `.geomod` file from the in-game menu.
2. **Sealed loader**: the loader's own code is stored encrypted (`core.sealed`) and only unsealed in memory, using a small native key module (`seal.node`).

> **What this is and isn't.** It stops casual reading, copying and editing: players can't open the files, tweak a locked mod's code or config, or read
> the loader's source. It is *not* unbreakable. The game runs JavaScript on the player's own computer, so the decryption key has to be reachable by the
> running program; someone willing to reverse-engineer the key module or attach a debugger can still get the code out. Treat it as a lock on the door, not a vault.
> If something must stay secret, keep it on a server instead.

## Locking a mod (for mod creators)
1. Make and test your mod as a normal folder in `mods\`.
2. Press **HOME**, open the **Create** tab, find your mod and click **Encrypt**.
3. The menu writes `mods\<id>.geomod` and (default, recommended) moves your original folder to `mods\_sources\<folder>` so the two don't both load.
4. Restart the game. The mod now shows **[locked]** in the Mods tab.
5. Share the single `.geomod` file. Keep your source folder safe: **only the package is shareable, only the folder is editable.**

What players can do with a locked mod: switch it on/off and change its **settings**. What they can't: see or change its code, its patches, its `config.json`
or any other file inside it.

If you edit your mod later, change the folder in `_sources` (move it back to `mods\`), bump `version`, and Encrypt again.

## Settings in the menu
A mod declares what players may adjust in `mod.json`:
```json
"settings": [
  { "key": "scale",  "type": "number", "label": "Speed scale", "min": 0.25, "max": 2, "step": 0.05, "default": 0.5, "help": "0.5 = half speed" },
  { "key": "mode",   "type": "select", "label": "Mode", "options": ["easy", "hard"], "default": "easy" },
  { "key": "debug",  "type": "bool",   "label": "Show debug info", "default": false },
  { "key": "nick",   "type": "text",   "label": "Name", "default": "", "maxLength": 40 }
]
```
In the menu, a mod with settings shows a **settings** link under its name. Values are validated by the loader (range, step, allowed options), stored in
`mods\_loader.json` under `settings`, and merged on top of your `config` defaults, so your transforms and `main.js` just read `ctx.config.scale` / `api.config.scale`.
Changes apply on the next restart (the menu shows the restart bar).

Locked mods can't use an editable `config.json`; use `settings` for anything the player may change, and `config` in `mod.json` for fixed defaults.

## What changes for mod code inside a package
* Everything works as before: `transforms`, `patches`, `main`, `renderer`, `css`, `requires`, routes.
* Files are read from memory, not from disk. In `main.js` use `api.readFile('data/x.json')` (returns a Buffer) instead of `fs.readFileSync(api.modDir + ...)`.
  `require('./helper')` between files of the same mod works; Node built-ins and Electron work too.
* `api.modDir` for a locked mod is a writable data folder, `mods\_data\<id>\`. Use it for anything the mod saves.
* To store a value the way the menu does, call `api.saveConfig({ key: value })` (this is how Game Tweaks remembers its multipliers).

## The package file
`mods\<id>.geomod` = magic `GEOMOD1`, a small public header (id, name, version, author, description, requires, settings: this is what the menu shows without
decrypting), then the encrypted contents: your files, compressed, sealed with AES-256-GCM. The key is derived per package from a random salt and the loader secret.
The header is authenticated too, so editing *anything* (even the version) makes the loader refuse the package: "damaged, edited, or made for a different loader".
A package opens on any copy of the loader built with the same secret.

## The sealed loader (for whoever builds the loader)
Files in `ModLoader\runtime\`:

| File | Role |
|---|---|
| `core.js` | tiny plain stub; the game's hook still loads this path. It loads the key module, unseals `core.sealed` in memory and runs it |
| `core.sealed` | the loader (core + menu UI), AES-256-GCM encrypted |
| `seal.node` | native Windows key module (a DLL). Holds the secret (masked) and only hands out derived keys; the secret never appears in JavaScript |
| `ui.js` | intentionally empty |

If the key module can't load, or `core.sealed` is damaged, the game just starts **without mods** and a line is written to `ModLoader\loader.log`.

**Crash guard.** The stub writes `runtime\.native-loading` just before loading the DLL and removes it right after. If the game crashed inside the DLL, the marker is left behind and the
next start skips the loader instead of crashing again. `python ModLoader\loader.py status` tells you, and `loader.py launch --retry-native` tries again.

**Windows DLL status.** The DLL was cross-compiled on Linux and its logic (key derivation, error handling) was tested as a Linux build of the same C source, matching Node's own HMAC byte for byte.
The Windows build itself has **not been run on Windows yet**. If the first launch shows no mod toast, check `loader.log` and `loader.py status`. Some antivirus tools distrust unsigned DLLs; an exception for the game folder may be needed.

## Rebuilding the sealed loader
Sources and tools live in `_dev\` (not part of what you distribute):
```
_dev\ModLoader-src\   core.js, ui.js, loader.py   <- edit these, never the sealed output
_dev\native\          seal.c.in (key module template), gen.js
_dev\build\           build.js, stub.tpl.js
_dev\private\         secret.hex   <- THE SECRET. Back it up. Never share it.
```
```
pip install ziglang
cd _dev\native
node gen.js ..\private\secret.hex                                   # writes seal.c + check.hex from the template
python -m ziglang cc -target x86_64-windows-gnu -shared -O2 -s -o seal-win.node seal.c
cd ..\build
node build.js ..\ModLoader-src ..\private\secret.hex ..\native ..\out   # -> out\core.js, core.sealed, seal.node
```
Copy `core.js`, `core.sealed` and `seal.node` from `_dev\build\..\out` into `ModLoader\runtime\`, and overwrite `ModLoader\runtime\ui.js` with a comment.

**The secret decides which packages open.** Keep the same `secret.hex` across rebuilds and every package ever made still works. Lose it, or generate a new one, and old
`.geomod` files can no longer be opened by new builds (re-pack them from your source folders).

## What is not sealed
`loader.py` and the launcher `.bat` are plain Python/batch: they only patch and start the game and contain no mod logic. The hook line in the game's `main.js` is visible by design.
Mods that ship as plain folders (everything in `mods\` before you click Encrypt) are plain files.
