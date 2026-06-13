# 01 — Root & File Structure

## File conventions

- **Extension:** `*.rmg.json`. A matching `*.png` of the same base name is the preview image
  shown in the in-game template picker.
- **Encoding:** UTF-8, plain JSON. The official files use tabs and are liberally formatted with
  blank lines; whitespace is irrelevant to the parser.
- **Location:** `<game install>\HeroesOldenEra_Data\StreamingAssets\map_templates\`.
- **Strict-ish JSON:** No comments or trailing commas appear in the official files. Assume
  standard JSON. Numbers may be integer or floating point; booleans are lowercase.

## Top-level object

The root is a single JSON object. Keys (in the order they conventionally appear):

| Key | Type | Required | Meaning |
|-----|------|:--------:|---------|
| `name` | string | ✔ | Internal template name. Often a localization key for the display name is **not** here — see `description`. |
| `gameMode` | string | ✔ | `"Classic"` or `"SingleHero"`. See below. |
| `description` | string | – | A **localization key** (e.g. `"templates_description_symmetry"`), not literal text. The game resolves it to localized copy. |
| `displayWinCondition` | string | – | A `win_condition_N` ID controlling the **label/icon** shown in the picker. The *actual* rules live in `gameRules.winConditions`. See [02](02-game-rules-and-win-conditions.md). |
| `sizeX` | int | ✔ | Map width in tiles. |
| `sizeZ` | int | ✔ | Map height in tiles. (`Z` is the second horizontal axis; maps are square in every official template, `sizeX == sizeZ`.) |
| `gameRules` | object | ✔ | Hero counts, bans, win conditions, start bonuses. See [02](02-game-rules-and-win-conditions.md). |
| `valueOverrides` | array | – | Per-object overrides of guard strength. See [04](04-content-and-placement.md#value-overrides). |
| `globalBans` | object | – | Banned items/spells. **Placement varies:** 19 official templates put it at the **root** (the editor model's location), 3 nest it **inside `gameRules`**. The game accepts both; root is the more common convention. See [02](02-game-rules-and-win-conditions.md#globalbans). |
| `variants` | array | ✔ | One or more whole-map layout alternatives. The generator picks one per game. See [03](03-variants-zones-connections.md). |
| `zoneLayouts` | array | – | Inline definitions of named zone layouts (terrain/obstacle/encounter profiles) referenced by zones. Many templates reference *built-in* layouts and omit or only partially populate this. See [03](03-variants-zones-connections.md#zone-layouts) and [04](04-content-and-placement.md). |
| `mandatoryContent` | array | – | Named groups of objects that are **guaranteed** to spawn in zones that reference them. See [04](04-content-and-placement.md#mandatory-content). |
| `contentCountLimits` | array | – | Named caps on how many of a given object SID may appear in a zone. See [04](04-content-and-placement.md#content-count-limits). |
| `contentPools` | array | – | Inline content-pool definitions. **Empty in every official template** — pools are referenced by ID and defined in the game's built-in data. Present for completeness/overrides. See [04](04-content-and-placement.md#content-pools). |
| `contentLists` | array | – | Inline content-list definitions. Also **empty in every official template**. See [04](04-content-and-placement.md#content-lists). |

### `gameMode`

- **`Classic`** — standard Heroes play; multiple heroes per player allowed
  (`gameRules.heroCountMin/Max` typically 4/8).
- **`SingleHero`** — one hero per player (`heroCountMin == heroCountMax == 1` in these templates).
  Used by duel/arena-style templates (e.g. *Symmetry*).

### `name` vs `description`

`name` is the raw identifier (e.g. `"Symmetry"`). `description` is a **localization key**
resolved by the game to localized flavour text. There is no literal human description string in
the template; the display name in-game is derived from localization tables keyed off the
template, not stored in the file.

---

## Annotated minimal skeleton

This is the smallest *structurally complete* shape (details elided with `…`). Each section links
to its detailed page.

```jsonc
{
  "name": "My Template",
  "gameMode": "Classic",
  "description": "templates_description_my_template",   // localization key
  "displayWinCondition": "win_condition_1",             // picker label only

  "sizeX": 96,
  "sizeZ": 96,

  "gameRules": {                                        // -> 02
    "heroCountMin": 4, "heroCountMax": 8, "heroCountIncrement": 1,
    "heroHireBan": false, "encounterHoles": false,
    "winConditions": { "classic": true, /* … */ },
    "bonuses": [ /* start-of-game bonuses -> 02 */ ]
  },

  "globalBans": { "magics": [ /* … */ ], "items": [ /* … */ ] },  // root (most common) -> 02

  "valueOverrides": [ /* { sid, variant, guardValue } -> 04 */ ],

  "variants": [                                         // -> 03
    {
      "orientation": { "mode": "MinimalBoundingSquare", /* … */ },
      "border":      { "obstaclesWidth": 3, "waterType": "water grass", /* … */ },
      "zones": [ /* zone objects -> 03 */ ],
      "connections": [ /* connection objects -> 03 */ ]
    }
  ],

  "zoneLayouts":        [ /* named terrain/encounter profiles -> 03/04 */ ],
  "mandatoryContent":   [ /* guaranteed object groups -> 04 */ ],
  "contentCountLimits": [ /* per-SID caps -> 04 */ ],

  "contentPools": [],   // always empty in official templates
  "contentLists": []
}
```

A complete, real, minimal-ish example is *Symmetry* (`../Templates/Symmetry.rmg.json`, ~700 lines)
— the smallest official template and a good reading starting point. A ready-to-edit probe
template is provided in [test-templates/](test-templates/).

---

## Map sizes

Sizes used by official templates are square. The editor groups them into labels:

| Size (tiles) | Label | Notes |
|---|---|---|
| 64 | S | |
| 80, 96 | M | |
| 112, 128 | L | |
| 144, 160 | XL | |
| 176, 192 | H (Huge) | |
| 208–256 | G (Giant) | |
| 272–512 | C (Colossal) | **[editor-inferred]** "experimental" sizes above the largest official size (240). Untested. |

The largest size appearing in official templates is **240**. Sizes above that (up to 512) are
offered by the editor as experimental and may not generate well. `sizeX` and `sizeZ` are stored
independently, so non-square maps are *structurally* expressible, but no official template uses
them — behaviour is unverified (see [06](06-open-questions-and-tests.md)).
