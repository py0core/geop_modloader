# Game Internals

Notes gathered while building the bundled mods. They describe the game version these mods were written against; names in the minified code change between versions,
but the data keys and file formats below have been stable.

> Everything here was read from the game's files. Items marked **(not tested in game)** were deduced from code, not verified by playing.

## Architecture
* Electron 42 app (Polytale Games, Steam app id 940230). Main process in `electron/`, UI in `renderer/app.bundle.js`, the simulation in
  `renderer/simulation-worker.bundle.js` (a Web Worker).
* The UI and the simulation exchange messages; most "game rules" live in the worker, most "what the player sees" in the app bundle.
* The game has its **own** Workshop mod system (`electron/mods/*`, with "families" for flags, political identities and presentation values). It is separate from this
  loader and only covers data like flags and names; we haven't explored it further.

## Saves (`%APPDATA%\Geoplanetical\saves\*.gsav`)
A `.gsav` is a container with a small header (metadata: nation, leader, in-game date, player country key) and several named **parts**:

| Part | Content |
|---|---|
| `core` | main game state |
| `countries` | per-country state: politics, parties, chambers, "social mind" |
| `people` | every named person: leaders, ministers, party figures |
| `construction`, `cgetrade`, `finance`, `transport`, `statistics` | the economy and logistics |

Each part is stored compressed (`zstd` or `deflate`) with a checksum. The game's own `electron/save-io.js` reads and writes the container
(`readMeta`, `unpackSave`, `packSave`, `hashBytes`); the Save Editor reuses it, so edited files are valid by construction. A 75 MB save opens in about a second.

Useful shapes (keys, not guaranteed forever):
* `countries[<countryKey>]` where the key is the ISO numeric code (`"376"` is Israel). `pol.parties[]` has `id`, `nameDesc {noun, qual}`, `color` (hsl), `ideology {econ, social, gov, nat}`,
  `support`, `cohesion`, `valence`, `organization {treasuryM, activistsK, influence, credibility}`.
  `pol.chambers[0]` has `seats` and `composition {partyId: seats}` which must add up to `seats`.
  `socialMind` holds `leaderPersonId` and `leadPartyId`.
* `people.people[]` has `id`, `name`, `given`, `surname`, `g`, `born`, `party`, `role`, `background`, `traits {charisma, integrity, authoritarian, competence, ambition, corruption, hawkish, health}`,
  `ideal {econ, social, gov, nat}`, `countryKey`.
* Party names shown to the player come from `nameDesc` (translated noun + qualifier), unless a custom name overrides them.

Safe-editing rules the Save Editor follows: never overwrite the original, only re-encode the parts you changed (the other parts stay byte-identical), verify checksums before the file becomes visible, and refuse edits that break invariants (chamber totals).
Editing saves may interact with the game's achievement/"campaign integrity" tracking **(not tested in game)**.

## Data tables (the tuning surface)
The simulation embeds large data literals with plain English/French key names. They are the easiest thing to mod. Found so far:

| Area | Keys to search for |
|---|---|
| Ground/naval/air movement | `movementProfiles`, `strategicKmpd`, `tacticalKmpd` |
| Air war | `commandReviewDays`, `supplyCapacityDays`, `airEngagementKm`, `profiles` (fighter, bomber, drone, attackHeli), `samRangeKmByGeneration`, `missileRangeKmByGeneration` |
| Unit stats | `attrs` with `attack`, `defense`, `armor`, `pen`, `rangeKm`, `speedKph`, `spotting` |
| Military economy | `personnel` (`trainCostPerRecruitK`), `stockpiles`, `baseTypes`, `forceStrategy` (AI arms growth) |
| Nuclear | `nuclear` (`warheadFabCostM`), `designs` (gun, implosion, boosted, thermo), `vectors` |
| Diplomacy | `relations` (`driftPerMonth`, `sanctionPenalty`, `allyAcceptRelation`), `choiceCoherence`, `aid`, `bloc` |
| Sanctions | `instruments` (`severity`, `selfCost`), `enforcement.levels` (`leakShare`) |
| Trade | `modes` (`speedKph`, `costPerLoadKm`), `cargo`, `passenger` |
| Disease | `diseases.<name>` (`r0`, `fatality`, `vaccine`) |
| Game speed | a table like `[0, .45, .9, 1.4]` (pause + three speeds) |

Keys starting with `_` (e.g. `_doc`) are the developers' French comments, and explain what a block does.

## Cablegram ("Le Fil")
The diplomacy messaging screen. Message text comes from translation keys; the player's options are **acts**: advice, law support, fund opposition, UN vote match, peace appeal, war join, compliment, insult.
Proposals become pending "cases" that the game resolves later. The view model exposes the current day, ISO date and player country key.
The `gemini-diplomatic-chat` mod hooks the thread renderer to add free text and AI replies.

## Things the game does *not* have (as far as the code shows)
* No unilateral "withdraw troops and release a territory/grant autonomy" action on Earth. Units get a `withdrawing` stance on their own, ceasefires and peace treaties
  (annex / cede / "only your withdrawal") are the only territorial mechanics, and "autonomous territory" status exists only for space colonies.
  A mod for this would need to edit province ownership and occupation state **(not investigated)**.
