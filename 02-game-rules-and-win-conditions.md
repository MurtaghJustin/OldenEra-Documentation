# 02 — Game Rules & Win Conditions

The `gameRules` object holds match-wide settings. It contains scalar rules, a `winConditions`
object, an optional `globalBans` object, and an optional `bonuses` array.

```jsonc
"gameRules": {
  "heroCountMin": 4,
  "heroCountMax": 8,
  "heroCountIncrement": 1,
  "heroHireBan": false,
  "encounterHoles": false,
  "tournamentRules": false,
  "factionLawsExpModifier": 1.0,
  "astrologyExpModifier": 1.0,
  "bonuses": [ /* start-of-game bonuses, see below */ ],
  "winConditions": { /* see below */ }
}
// NOTE: globalBans usually lives at the ROOT, beside gameRules — not inside it. See below.
```

## Scalar rules

| Field | Type | Meaning |
|-------|------|---------|
| `heroCountMin` | int | Minimum heroes a player may field. `Classic` templates use 4; `SingleHero` uses 1. |
| `heroCountMax` | int | Maximum heroes. `Classic` uses 8; `SingleHero` uses 1. |
| `heroCountIncrement` | int | Per-castle increase to the hero cap **[editor-inferred]** — extra heroes a player may hire per town owned. Usually 1. |
| `heroHireBan` | bool | If true, heroes cannot be hired from taverns (army of starting hero only). |
| `encounterHoles` | bool | Enables "encounter holes" — pits/gaps within encounter footprints. Interacts with per-zone `encounterHolesSettings` (see [03](03-variants-zones-connections.md#zone-fields)). |
| `tournamentRules` | bool | Enables tournament-mode rule set. Pairs with `winConditions.tournament`. |
| `factionLawsExpModifier` | double | Multiplier on experience gained from Faction Laws progression. `1.0` = default. |
| `astrologyExpModifier` | double | Multiplier on experience from Astrology. `1.0` = default. |

## `winConditions`

A flag-and-parameter bag. Multiple conditions can be enabled simultaneously; the meaningful set
varies by template. Booleans turn a condition on; the adjacent `*Day`/`*Value`/`*Days` fields
parameterise it.

| Field | Type | Meaning |
|-------|------|---------|
| `classic` | bool | Standard "defeat all opponents" victory. Nearly always `true`. |
| `desertion` | bool | Enables an army-desertion mechanic. |
| `desertionDay` | int | Day desertion checks begin. |
| `desertionValue` | int | Threshold (army value) governing desertion. |
| `heroLighting` | bool | Enables "hero lighting" (spelled this way in data). **[unknown — see 06]** likely a hero highlight/visibility or a lightning-strike loss condition. |
| `heroLightingDay` | int | Day the above activates. |
| `lostStartCity` | bool | Player loses if their starting city is captured. |
| `lostStartCityDay` | int | Grace day before that applies. |
| `lostStartHero` | bool | Player loses if their starting hero dies. |
| `cityHold` | bool | A designated city must be **held** for `cityHoldDays` to win. Pairs with a main object flagged `holdCityWinCon`. |
| `cityHoldDays` | int | Days the hold-city must be retained. |
| `gladiatorArena` | bool | Enables the Gladiator Arena win path. |
| `gladiatorArenaRegistrationStartWork` | bool | Whether registration opens at "start work". |
| `gladiatorArenaRegistrationStartFight` | bool | Whether registration opens at "start fight". |
| `gladiatorArenaDaysDelayStart` | int | Days before the arena opens (e.g. 30). |
| `gladiatorArenaCountDay` | int | Arena combat cadence in days (e.g. every 3). |
| `championSelectRule` | string | Only known value: `"StartHero"`. |
| `tournament` | bool | Enables tournament victory. |
| `tournamentDays` | int[] | Days on which tournament rounds occur. |
| `tournamentAnnounceDays` | int[] | Days rounds are announced ahead of time. |
| `tournamentPointsToWin` | int | Points required to win the tournament. |
| `tournamentSaveArmy` | bool | Whether the army is preserved between tournament fights. |

### `displayWinCondition` vs `winConditions`

`displayWinCondition` (a root field, see [01](01-root-and-structure.md)) only selects the
**label/icon** in the template picker. The mechanically-enforced rules come from the
`winConditions` object. Known display IDs:

| ID | Editor label | Used by (count) |
|----|--------------|-----------------|
| `win_condition_1` | Standard | 29 |
| `win_condition_3` | Lost Starting City | 18 |
| `win_condition_4` | Gladiator Arena **[editor-inferred; label commented out in source]** | 4 |
| `win_condition_5` | Hold City | 5 |
| `win_condition_6` | Tournament | 5 |

`win_condition_2` does not appear in any official template and its label is unknown
(see [06](06-open-questions-and-tests.md)).

## `globalBans`

Removes specific artifacts and spells from the whole map.

> **Placement.** Despite the heading here, `globalBans` is most often a **root-level** field
> (19 of 22 templates that use it), sitting alongside `gameRules` rather than inside it; the other
> 3 nest it inside `gameRules`. Both work. The example below shows the object's shape regardless of
> where it sits.

```jsonc
"globalBans": {
  "magics": [ "neutral_magic_town_portal", "neutral_magic_dimension_door", … ],  // spell SIDs
  "items":  [ "pole_star_artifact", "seven_league_boots_artifact", … ]           // artifact SIDs
}
```

- `magics` — spell SIDs (see [05 — Spells](05-id-reference.md#spells)). Commonly banned: the
  five movement/teleport "neutral" spells (Town Portal, Dimension Door, Gate of Light,
  Shadowflight, Pocket Dimension) in duel templates.
- `items` — artifact SIDs (see [05 — Bannable artifacts](05-id-reference.md#bannable-artifacts)).
  Commonly banned: movement boots, diplomacy artifacts, war-suppression artifacts.

## `bonuses` — start-of-game effects

Each entry grants something to heroes at game start.

```jsonc
{
  "sid": "add_bonus_res",          // bonus type
  "receiverSide": -1,              // -1 = all sides/players
  "receiverFilter": "start_hero",  // "start_hero" or "all_heroes"
  "parameters": [ "gold", "10000" ]
}
```

| `sid` | Effect | `parameters` shape |
|-------|--------|--------------------|
| `add_bonus_res` | Starting resources | `[ <resource>, <amount> ]` where resource ∈ `gold, wood, ore, mercury, crystals, gemstones` |
| `add_bonus_hero_spell` | Grant a spell | `[ <spellSid> ]` |
| `add_bonus_hero_stat` | Modify a hero stat / spell cost | e.g. `[ "movementBonus", <amt> ]` or `[ "magicCostSidSet", <spellSid>, "-999", "0" ]` to make a spell free |
| `add_bonus_hero_item` | Grant an artifact | `[ <artifactSid> ]` |
| `add_bonus_hero_unit_multipler` | Multiply starting army | `[ <multiplier> ]` *(note: SID is misspelled "multipler" in the data — copy it verbatim)* |

`receiverFilter`: `"start_hero"` (only the starting hero) or `"all_heroes"` (every hero a player
gets). `receiverSide`: `-1` targets all players.

**Composite example — "free Town Portal for the starting hero"** (two bonuses: grant the spell,
then set its cost to free):

```jsonc
{ "sid": "add_bonus_hero_spell", "receiverSide": -1, "receiverFilter": "start_hero",
  "parameters": [ "neutral_magic_town_portal" ] },
{ "sid": "add_bonus_hero_stat",  "receiverSide": -1, "receiverFilter": "start_hero",
  "parameters": [ "magicCostSidSet", "neutral_magic_town_portal", "-999", "0" ] }
```
