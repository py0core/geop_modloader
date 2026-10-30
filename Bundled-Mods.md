# Bundled Mods

| Mod | Key | What it does | How it works |
|---|---|---|---|
| **Slower Time** `slow-time` | | Scales all game speeds. Edit `scale` in its `config.json` (0.5 = half). | Transform multiplies the speed table in the app and the simulation worker. |
| **Game Tweaks** `game-tweaks` | **F9** | ~50 multipliers for hidden values: unit/ship speeds, ranges, supply, costs, nuclear, diplomacy, sanctions, disease, game speed. Presets included. Applies on restart. | `catalog.json` lists anchors per setting; `tweak.js` multiplies the numbers; `panel.js` is the UI; `main.js` saves `config.json`. |
| **Save Editor** `save-editor` | **F8** | Edit a save's parties (name parts, colour, ideology, support, seats, treasury) and leaders (names, birth year, traits, ideals). Always writes a new `<save>-edited` file. | `engine.js` uses the game's `save-io.js`; `editor.js` is the panel; routes connect them. |
| **Gemini Core** `gemini-core` | | Shared connection to Google Gemini. Holds your API key. | Local service on `127.0.0.1:47831` in the main process. |
| **Gemini AI Diplomacy** `gemini-ai-diplomacy` | | Gemini decides AI foreign-policy plans. | Patches the simulation worker; log `gemini-decisions.jsonl`. |
| **Gemini Diplomatic Chat** `gemini-diplomatic-chat` | | Text box under Cablegram threads. Write in your own words; replies are in character and can trigger real game actions. | 5 transforms on the app bundle plus `chat.js` / `chat.css`; log `gemini-chat.jsonl`. |

All of them can be locked into `.geomod` packages from the menu's Create tab and still work (tested with every bundled mod).

## Gemini setup
Create `%APPDATA%\Geoplanetical\gemini.json` (the core mod creates a template on first run):
```json
{ "apiKey": "YOUR_KEY", "model": "gemini-3.8-flash" }
```
Other keys: `chatEnabled` (false turns the chat off), `fallbackModels`, `chatTimeoutMs`, `chatHedgeMs`, `chatThinkingLevel`.
Free-tier keys are rate limited; if replies time out or return 503 the model is overloaded, and the mod falls back to other models and finally to the game's stock text.

## Using them as examples
* Smallest transform: `slow-time`
* Lots of numeric edits with safe anchors: `game-tweaks`
* Main-process logic + a UI panel + routes: `save-editor`
* Rewriting UI code and injecting scripts/CSS: `gemini-diplomatic-chat`
* A shared background service used by several mods: `gemini-core`
