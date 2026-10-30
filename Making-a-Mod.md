# Making a Mod

A mod is a folder in `mods\` with a `mod.json`. Nothing else is required.

## Smallest possible mod
`mods\hello\mod.json`
```json
{ "id": "hello", "name": "Hello", "version": "1.0.0", "loaderApi": 1,
  "description": "Says hello in the console.", "renderer": ["hello.js"] }
```
`mods\hello\hello.js`
```js
console.log('hello from my mod');
```
Restart the game: Hello appears in the HOME panel, and the message is in the game's developer console.

## mod.json reference
| Field | Type | Meaning |
|---|---|---|
| `id` | string | **Required.** Lowercase letters, digits, hyphens, 2-48 chars. Must be unique. |
| `name`, `version` | string | **Required.** Shown in the panel. |
| `loaderApi` | number | **Required.** Always `1` for now. |
| `author`, `description` | string | Shown in the panel. |
| `requires` | string[] | Ids of mods that must load first (e.g. `["gemini-core"]`). |
| `config` | object | Default settings. Overridden by `config.json` in the mod folder. |
| `patches` | array | Find-and-replace edits on game files. See [Patching the Game](Patching-the-Game.md). |
| `transforms` | array | Scripts that rewrite a game file. Same page. |
| `main` | string | Script that runs in Electron's main process. See [Main Process, Routes and UI](Main-Process-Routes-and-UI.md). |
| `renderer` | string or string[] | Scripts injected into the game window. |
| `css` | string or string[] | Stylesheets injected into the game window. |
| `settings` | array | Options players can change in the menu, even for locked mods. See [Locking and Sealing](Locking-and-Sealing.md). |
| `csp` | object | Extra Content-Security-Policy sources, e.g. `{ "connect-src": ["http://127.0.0.1:1234"] }`, if your page needs to reach a local service. |

Paths are relative to the mod folder; absolute paths and `..` are rejected.

## Settings (`config`)
Put defaults in `mod.json` under `"config"`. Users (or your own mod) override them in `config.json` next to it; the two are merged.
Transforms receive the merged object as `ctx.config`; `main` receives it as `api.config`.

## Sharing without your source
Press HOME > **Create** > **Encrypt** to turn the folder into one locked `.geomod` file. See [Locking and Sealing](Locking-and-Sealing.md).

## Rules the loader enforces
* A bad `mod.json` marks the mod **failed** with the reason in the panel; other mods still load.
* If *any* patch of a mod doesn't match, the **whole mod** is skipped (nothing of it is applied).
* Patches and transforms may only touch `renderer/*` files, and are never written to disk.
* Enabled flags live in `mods\_loader.json`; a changed flag takes effect after a restart.

## Worked example: slow-time
`mod.json`
```json
{ "id": "slow-time", "name": "Slower Time", "version": "1.0.0", "loaderApi": 1,
  "config": { "scale": 0.5 },
  "transforms": [ { "file": "renderer/simulation-worker.bundle.js", "script": "scale.js" },
                  { "file": "renderer/app.bundle.js",               "script": "scale.js" } ] }
```
`scale.js` finds the game's speed table (`var X=[0,.45,.9,1.4];`) and multiplies the three speeds by `scale`. It throws if the table isn't found exactly
once, which is how a mod says "the game changed, I need updating".
