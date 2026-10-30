# Main Process, Routes and UI

Besides patching code, a mod can run its own code in two places.

## The main process (`"main": "main.js"`)
Electron's main process has full Node.js: files, network, child processes. Use it for anything the game window shouldn't do itself
(saving files, calling web APIs with a secret key, reading/writing saves).
```js
module.exports = {
  start(api) {
    // api = { app, electron, loaderVersion, modId, modDir, rootDir, config, userData, log, route }
    api.log('hello');
  },
};
```
| `api.` | Meaning |
|---|---|
| `app`, `electron` | Electron's `app` and the module itself |
| `modDir`, `rootDir` | your mod folder (for a locked mod: a writable data folder `mods\_data\<id>`), the game folder |
| `userData` | `%APPDATA%\Geoplanetical` (saves, settings, logs) |
| `config` | merged `config` from `mod.json` + `config.json` |
| `log(...)` | writes to `ModLoader\loader.log` with your mod id |
| `route(name, fn)` | register a function the window can call (below) |
| `readFile(rel)` | read a file of your own mod as a Buffer (works for folders **and** locked packages; prefer this over `fs` + `modDir`) |
| `saveConfig(obj)` | store values the way the menu's settings do; they come back in `config` after the next restart |

Main-process code runs with the user's permissions, so treat other people's mods like programs you downloaded.

## Routes: window to main process
Register in `main.js`:
```js
api.route('save', async body => { /* do work */ return { saved: 3 }; });
```
Call from your injected renderer script:
```js
const G = window.__GEOMOD;            // { port, token, hotkey, ... } provided by the loader
const r = await fetch(`http://127.0.0.1:${G.port}/mod/<your-mod-id>/save`, {
  method: 'POST',
  headers: { 'x-geomod-token': G.token, 'content-type': 'text/plain' },   // text/plain avoids a browser CORS preflight
  body: JSON.stringify({ hello: 1 }) });
const j = await r.json();             // { ok:true, result } or { ok:false, error }
```
Body limit 4 MB. Route names: letters, digits, hyphen. The token changes every launch, so other programs and web pages can't call it.
Restart the game from your UI with `POST /restart` (same header).

## Building a panel
The bundled Save Editor (`mods/save-editor/editor.js`) and Game Tweaks (`panel.js`) are good starting points. Their recipe:

* Create a host `<div>`, `position:fixed; inset:0; pointer-events:none`, and attach a **Shadow DOM** (`attachShadow({mode:'open'})`).
  Your CSS can't leak into the game and the game's CSS can't leak in.
* Put the visible panel inside with `pointer-events:auto`.
* **Stop keyboard and mouse events at the host** (`host.addEventListener('keydown', e => e.stopPropagation())`, and likewise `keyup`, `keypress`,
  `wheel`, `pointerdown`, `mousedown`, `click`...). Otherwise typing in a text box also triggers the game's shortcuts and clicks reach the map.
* Open it with your own key via a capturing `keydown` listener on `window` (`F8` and `F9` are taken by the bundled mods, `HOME` by the loader).
* Wrap `localStorage` in try/catch if you use it; it can be unavailable.
* Style: the loader's panel imitates ReShade/Dear ImGui (monospace, `#29487a` title bar, `rgba(66,150,250,...)` buttons). Reusing those colours keeps mods consistent.

## Talking between mods
* `requires` only controls **load order**: your mod loads after the listed mods, and fails with a clear message if one of them is missing or failed.
  To share data or behaviour, use a local HTTP service in `main` or a prefixed global in the window.
* The Gemini mods share one background service (`gemini-core`) on `127.0.0.1:47831`; the others call it with `fetch`. Reuse that rather than adding another.
* Renderer scripts of different mods share one `window`, so expose a small object (the chat mod uses `self.__GEMCHAT`) and keep names prefixed.

## Quick sanity test without launching the game
Because mods are plain JS, you can test transforms in Node on a copy of the extracted bundle (feed `text` in, check the result with `vm.Script`), and test panels
in `jsdom` with a fake `window.__GEOMOD` and a tiny HTTP server that implements your routes. The bundled mods were tested this way. WebGL, and
therefore the actual map, can't run in such tests, so always do a final check in the real game.
