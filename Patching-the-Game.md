# Patching the Game

The game's logic lives in a few big, minified JavaScript files inside `resources\app.asar` (a 3.6 GB archive). Mods never edit it;
they ask the loader to hand the game a modified copy of a file while it loads.

## Which file holds what
| File (inside the archive) | Size | What is in it |
|---|---|---|
| `renderer/app.bundle.js` | ~31 MB | The whole UI plus shared game code and translations |
| `renderer/simulation-worker.bundle.js` | ~97 MB | The simulation: economy, war, diplomacy, politics, data tables |
| `renderer/war-theatre-worker.bundle.js` | ~1.4 MB | War theatre helper |
| `renderer/save-codec-worker.bundle.js`, `world-market-worker.bundle.js`, `city-lights-worker.bundle.js` | | Save codec, markets, night-lights rendering |
| `electron/*.js` | | Main process: saves (`save-io.js`), windows, the game's own mod system (`electron/mods/`) |

Patches and transforms may only target `renderer/...` files. Worker files are supported: the loader writes the patched worker to
`ModLoader\cache\` and redirects the game to it.

## Reading the code
Pull a single file out of the archive (extracting the whole archive is not necessary and may fail on a patched header):
```
npx @electron/asar extract-file "<game folder>\resources\app.asar" renderer/app.bundle.js
```
The code is minified (`function a3B(A,B){...}`), but **string keys survive**: property names, translation keys (`"war.occupation.title"`) and
data-table field names are readable. Search for those, not for function names. Function and variable names such as `Ro` or `YDB` change between
game versions, so a mod must never depend on them.

Useful search ideas: a word from the UI text (find its translation key, then search for the key), a property name (`driftPerMonth`),
or a number you can see in game.

## Option 1: a simple patch
```json
"patches": [ { "file": "renderer/app.bundle.js",
               "find": "maxRoster:320", "replace": "maxRoster:640" } ]
```
| Field | Meaning |
|---|---|
| `find` / `replace` | exact text (or a regex if `"regex": true`, with optional `"flags"`) |
| `all` | replace every occurrence (otherwise `find` must match **exactly once**) |
| `count` | require exactly this many matches |
| `optional` | don't fail the mod if the text is absent |

If the count is wrong the mod is marked **failed** and applies nothing. That is intentional: after a game update, a stale mod stops
instead of corrupting something.

## Option 2: a transform script
When you need logic, write a script that receives the whole file and returns the changed text:
```js
module.exports = (text, ctx) => {
  // ctx = { mod, config, file, enc, args, readModFile }
  const out = text.replace(/maxRoster:(\d+)/, (m, n) => 'maxRoster:' + n * ctx.config.factor);
  if (out === text) throw new Error('anchor not found - game updated?');
  return out;
};
```
Register it under `"transforms": [ { "file": "renderer/simulation-worker.bundle.js", "script": "t.js" } ]`.
`ctx.readModFile('catalog.json')` reads a file from your mod folder.

Good habits:
* **Throw when the anchor isn't found** (once) so failures are loud.
* Anchor on *keys and structure* (`foot:{strategicKmpd:`) rather than on names the minifier picked.
* A game value may appear in both the UI bundle and the simulation worker; patch both if they must agree.
* Check your output at least parses: `new (require('vm').Script)(patchedText)` in Node.

## Data tables: the easy wins
The simulation keeps its tuning numbers in large data literals. Changing a number there is usually safe and very effective.
Examples (all verified): unit march speeds (`movementProfiles`), air and missile ranges, supply, diplomacy (`relations`), sanctions, disease,
nuclear programs. The **Game Tweaks** mod is built entirely on this idea: `catalog.json` lists, for each setting, the exact text that
precedes each number, and `tweak.js` multiplies them. Read that mod as a template for "change many numbers safely":

1. all matches are located on the **original** text first, then applied together (so edits can't affect each other's anchors);
2. it refuses overlapping edits;
3. missing anchors are skipped with a warning rather than breaking the game.

## After a game update
Launch with the `.bat`. Mods whose anchors still match keep working; the rest show **failed** with the reason. Re-extract the file, search for
the key you anchored on, and update the `find` text.
