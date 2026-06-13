# 03 — Variants, Zones & Connections

This is the geometric heart of a template: the zones, how they're placed, and how they're linked.

## `variants`

`variants` is an array of complete alternative map layouts. **The generator selects one variant
per game.** Most official templates have a single variant; a few use several (e.g. *Anarchy* has
5, *Arcade* / *Jebus Outcast* / *Maze* have 3) to add layout variety.

Each variant object:

```jsonc
{
  "orientation": { … },   // how zones are placed/rotated on the grid
  "border":      { … },   // map-edge obstacle/water framing
  "zones":       [ … ],   // the regions
  "connections": [ … ]    // graph edges between zones
}
```

### `orientation`

Controls how the zone graph is positioned and rotated on the square grid. **[Most fields editor-inferred]**

| Field | Type | Meaning |
|-------|------|---------|
| `mode` | string | Packing strategy. `"MinimalBoundingSquare"` (44 uses) fits zones into the smallest axis-aligned square; `"BoundingCircle"` (12 uses) fits them in a circle. |
| `zeroAngleZone` | string | Name of the zone used as the angular reference (0°). The layout is rotated so this zone sits at the reference angle. |
| `baseAngleMin` / `baseAngleMax` | double | Range for the starting angle, **in degrees** in the official files (e.g. `0`–`360`). |
| `randomAngleAmplitude` | double | Random jitter (± degrees) added to placement angles to avoid perfect symmetry (e.g. `45`). |
| `randomAngleStep` | double | Quantisation step for angles in degrees (e.g. `90` snaps to right angles). |

> Note: the editor's source comments describe these angles in *radians*, but the official files
> use **degree-valued** ranges (`baseAngleMax: 360`, `randomAngleStep: 90`). Treat the official
> files as authoritative and use degrees.

### `border`

Frames the map edge with obstacles and/or water.

| Field | Type | Meaning |
|-------|------|---------|
| `cornerRadius` | double | Rounds the map corners (fraction, e.g. `0.15`). |
| `obstaclesWidth` | int | Width (tiles) of the impassable obstacle band around the edge. |
| `obstaclesNoise` | array | `[{ "amp": <double>, "freq": <double> }]` — perturbs the obstacle band edge with noise of given amplitude/frequency. |
| `waterWidth` | int | Width (tiles) of a water band; `0` disables. |
| `waterNoise` | array | Same shape as `obstaclesNoise`, for the water band. |
| `waterType` | string | Only known value: `"water grass"`. |

---

## Zones

Each entry in `variant.zones` is a region. Below is the **complete field reference**; not all
fields appear in every zone.

### Identity & shape

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | Unique zone name within the variant; referenced by connections and roads (e.g. `"Spawn-A"`, `"Side-A1"`). |
| `size` | double | **Relative** area weight. `1.0` is the baseline; `0.7` smaller, `1.25` larger. Controls the zone's share of the map and (proportionally) its content budget. |
| `layout` | string | Name of a zone layout (terrain/obstacle/encounter profile). May reference a built-in layout or one defined in the root `zoneLayouts`. See [layout list](#zone-layouts). |
| `crossroadsPosition` | int | `0` = no central road junction; `1` = the zone has a **crossroads** (a central node roads/connections fan out from). Roads can target `{ "type": "Crossroads" }`. |
| `diplomacyModifier` | double | Adjusts neutral-creature join/diplomacy odds in this zone. Observed values: `-0.25`, `-0.5` (negative = harder to recruit neutrals). |

### Guard tuning

These shape how strongly content inside the zone is defended.

| Field | Type | Meaning |
|-------|------|---------|
| `guardCutoffValue` | int | **Minimum guard value to keep (confirmed).** Any content guard whose computed value falls below this is **dropped entirely** — the object is left unguarded. A cutoff set above the zone's content guard values removes *all* guards (the zone becomes fully unguarded). Hence the common `1500` strips trivial sub-1500 guards. Observed: 0, 1000, 1500 (most common), 2000–3500. |
| `guardRandomization` | double | Per-guard random ± fraction applied to guard values (confirmed: at `0.25`, six identical objects had clearly varied guard sizes vs. uniform sizes at `0.0`). So `0.05` = ±5%. Official templates use `0.05`–`0.25`; **stay within that range** — a test at `0.5` (double the max) caused guarded objects to go missing from the zone. |
| `guardMultiplier` | double | Scales the zone's **content** guard values (confirmed: a ×2.0 zone's guards were far stronger than an otherwise-identical ×0.5 zone). Does **not** affect border/connection guards. E.g. `0.85`, `1.0`. |
| `guardWeeklyIncrement` | double | **Compounding** weekly growth fraction (confirmed): each week the guard value is multiplied by `(1 + increment)`. With `1.0` the guard **doubled every week** (1× → 2× → 4×). So `0.10` = +10% compounding per week. The same field on connections and main objects behaves the same way. |
| `guardReactionDistribution` | int[6] | Six weights distributing the zone's **content/object guards** across six **disposition** tiers (index `0` = least aggressive → index `5` = friendliest). The *realised* reaction still resolves against relative army strength as normal: index-0 guards **flee only when the hero overwhelmingly outmatches them, and fight otherwise**; index-5 guards **offer to join**. Sets disposition, not guard strength, and does **not** affect border/connection guards (those always fight). Middle indices `1`–`4` unconfirmed. Examples: `[60,20,10,5,2,0]`, `[1,1,4,4,2,1]`, `[3,2,0,0,0,0]`. |

> **Guard value is an army-*value* budget (confirmed in-game).** A guard's `guardValue` (here and
> on connections/main objects) is an abstract value the generator fills with creatures appropriate
> to the **local biome/faction**, not a fixed creature count. Two connections both set to
> `guardValue: 10000` produced *different* stacks — 20–49 Grolls (T3 neutral) on one border vs.
> 50–99 Stinging Ra'Shoths (upg. T1) + 20–49 Votaries (upg. T2 Schism) on the other: same budget,
> different composition, counts scaling inversely with creature tier. So identical values give
> comparable *strength* but varied armies, and nearby zones' biome/faction theme drives which
> creatures appear.
>
> **Scaling is linear, realized via creature tier (confirmed).** A 10× larger budget produces a
> ~10×-value army by selecting a **higher-tier creature at a similar count**, not 10× more bodies:
> a `5000` border gave 20–49 T1 Parasites while a `50000` border on the same map gave 20–49
> **upgraded-T7 Vampire Lords** (same count band, ~10× per-head value). Once a single top-tier stack
> tops out, larger budgets add **more stacks**: a `200000` guard appeared as **two** 20–49 T7 stacks
> (one upgraded, one base).

> **`guardReactionDistribution` is a friendliness/disposition ladder for the zone's content guards
> (confirmed in-game).** Each guard is assigned a tier drawn from the six weighted buckets
> (index `0` = least friendly → index `5` = friendliest). The tier does **not** set guard strength,
> and **border/connection guards ignore it entirely — they always Fight.**
>
> The game only ever displays three reaction values — **Will Fight / Will Flee / Will Join** —
> viewable via a reveal spell and computed for the **viewing hero's army**. How a tier resolves to
> one of those three:
> - **Strength matters:** the more a hero outpowers a guard, the more it shifts away from *Fight*
>   toward *Flee* (no Diplomacy) or *Join* (with Diplomacy). A weak hero gets *Fight* at any tier.
> - **Diplomacy gates joining for tiers 0–4:** with the **Diplomacy** skill, guards on these tiers
>   can *Join* once the hero's army is overwhelming enough (a higher tier joins at a smaller
>   advantage). **Without Diplomacy, tiers 0–4 never Join** — only Fight or Flee — *no matter how
>   large the army.*
> - **Tier 5 is the "free-join" tier:** index-5 guards offered to **Join even without Diplomacy**.
>
> So a guard's tier governs *how readily it flees/joins* and *whether joining needs Diplomacy*, while
> the final Fight/Flee/Join is computed against the viewing hero (army + Diplomacy). This also
> explains why official templates weight the low indices (e.g. `[60,20,10,5,2,0]`) — those are
> ordinary guards that fight unless heavily outmatched and won't join a Diplomacy-less hero.
> *(Still unmapped: the exact army-advantage threshold separating tiers 1–4 from one another.)*

### Content budgets & pools

Each zone draws content from three pools, each with a value budget. Pools are referenced by ID
(definitions live in built-in game data — see [04](04-content-and-placement.md)).

| Field | Type | Meaning |
|-------|------|---------|
| `guardedContentPool` | string[] | Pool(s) of **guarded** objects (banks, dwellings, strong buildings). |
| `unguardedContentPool` | string[] | Pool(s) of freely-accessible objects. |
| `resourcesContentPool` | string[] | Pool(s) of resource pickups/mines. |
| `guardedContentValue` | int | Total value budget of guarded content (absolute). |
| `guardedContentValuePerArea` | int | Value budget **per unit area** (scales with zone size). |
| `unguardedContentValue` / `unguardedContentValuePerArea` | int | As above, for unguarded content. |
| `resourcesValue` / `resourcesValuePerArea` | int | As above, for resources. |
| `mandatoryContent` | string[] | Names of `mandatoryContent` groups guaranteed in this zone. |
| `contentCountLimits` | string[] | Names of `contentCountLimits` rules applied to this zone. |

> **The value budgets drive pool content; `0` means none (confirmed).** A zone with
> `guardedContentValue: 0` **and** `guardedContentValuePerArea: 0` but a guarded pool referenced
> produced **no guarded content at all** — only its mandatory/explicit objects appeared. This is how
> *Symmetry* (all value fields `0`) works: the pools contribute nothing and only mandatory content
> spawns. So the absolute and per-area budgets are what pull content from a pool; a referenced pool
> with a zero budget is inert.
>
> **Either budget alone works (confirmed).** `guardedContentValue` alone produces guarded content;
> `guardedContentValuePerArea` alone *also* produces it, **scaling with zone area** (a `perArea: 2000`
> zone yielded more guarded content than a separate `value: 150000` zone). So they're two independent
> ways to fund a pool. (Tested for guarded; the unguarded/resource budgets behave the same way. Still
> untested: how an absolute *and* a per-area value combine when **both** are non-zero — add, or one
> overrides — see [06 Q5](06-open-questions-and-tests.md).)

### Biome selectors

Three selectors decide the terrain/biome of the zone, its content, and its meta-objects.
All three use the **selector grammar** below.

| Field | Selects the biome of… |
|-------|------------------------|
| `zoneBiome` | the zone terrain itself |
| `contentBiome` | content objects placed in the zone |
| `metaObjectsBiome` | meta/decoration objects |

```jsonc
{ "type": "<SelectorType>", "args": [ … ] }
```

| Selector `type` | Meaning | `args` |
|-----------------|---------|--------|
| `FromList` | Pick from an explicit biome list (random if several, any if empty). | Biome names, e.g. `["Sand"]`, `["Grass","Lava","Snow","Dirt","Deathland","Autumn"]`, or `[]` (any). Also supports `["differentFrom: <ZoneName>"]` to force a biome different from another zone's. |
| `MatchMainObject` | Match the biome of one of the zone's main objects. | `["0"]` = first main object (the spawn/city). |
| `MatchZone` | Match another zone's biome. | a zone name. |
| `Match` | Generic match. | `["0"]` etc. (rare for biomes; common for `faction`). |

**Known biome names:** `Grass`, `Lava`, `Snow`, `Dirt`, `Deathland`, `Autumn`, `Sand`.
(Confirmed from `FromList` args across templates; see [05](05-id-reference.md#biomes).)

> **Biome names map to terrains, and an explicit `FromList` biome wins over faction-native terrain
> (confirmed in-game).** Forcing `zoneBiome: FromList ["Grass"]` / `["Snow"]` / `["Lava"]` produced
> exactly grassland / snow / lava zones — *regardless* of the town faction's normal native terrain
> (both forced spawns sat on non-native ground for their factions). By contrast,
> `MatchMainObject` makes the zone terrain **follow** the main object's faction. So: use an explicit
> `FromList` biome to override terrain; use `MatchMainObject`/`MatchZone` to derive it.

### `mainObjects`

The "anchor" objects of a zone — spawns, cities, arenas. Each:

```jsonc
{
  "type": "City",
  "spawn": "Player1",                 // only for type "Spawn"
  "owner": null,
  "guardChance": 0.5,                 // probability this object is guarded
  "guardValue": 5000,                 // guard strength if guarded
  "guardWeeklyIncrement": 0.10,
  "removeGuardIfHasOwner": true,      // drop guard when the object has an owner (e.g. the player's own spawn)
  "buildingsConstructionSid": "poor_buildings_construction",  // pre-built building set for cities
  "faction": { "type": "Match", "args": ["0"] },              // selector, see below
  "placement": "Uniform",             // where in the zone it goes
  "placementArgs": [],
  "holdCityWinCon": false             // true => this city is the Hold-City objective
}
```

| Field | Meaning |
|-------|---------|
| `type` | `Spawn` (player start), `City` (town/castle), `AbandonedOutpost`, `GladiatorArena`. |
| `spawn` | For `Spawn` type: the player slot, `"Player1"`…`"Player8"`. |
| `guardChance` | 0–1 probability the object is guarded. |
| `guardValue` | Guard army strength if guarded. |
| `removeGuardIfHasOwner` | If true, no guard once the object is owned (typical for the owning player's spawn). |
| `buildingsConstructionSid` | Which pre-construction building loadout a town starts with (poor → ultra-rich, plus template-specific sets). See [05](05-id-reference.md#building-construction-sids). |
| `faction` | Selector choosing the town's faction (see grammar below). |
| `placement` | `Uniform` (anywhere in zone), `Center`, `Connection` (near a connection), `NearZone`. |
| `placementArgs` | Extra args for placement; usually empty. |
| `holdCityWinCon` | Marks this city as the Hold-City win objective (pairs with `winConditions.cityHold`). |

#### `faction` selector grammar

Same `{ type, args }` shape as biomes. Common patterns observed:

- `{ "type": "Match", "args": ["0"] }` — match faction of main object index 0 (e.g. the spawn).
- `{ "type": "Match", "args": ["0", "Spawn-B"] }` — match the faction used at index 0 of zone `Spawn-B` **[interpretation]**.
- `{ "type": "FromList", "args": [] }` — any faction.
- `{ "type": "FromList", "args": ["differentFrom: 0"] }` — any faction different from index 0.
- `{ "type": "FromList", "args": ["differentFrom: 0 Spawn-A", "differentFrom: 0 Spawn-B"] }` —
  different from the factions chosen in `Spawn-A` and `Spawn-B`.

The `differentFrom: <index> <ZoneName>` token references the faction selected for main-object
`<index>` of `<ZoneName>`. This is how templates keep neutral towns distinct from player factions.

### `roads`

Per-zone road segments connecting endpoints inside/through the zone.

```jsonc
"roads": [
  { "type": "Stone", "from": { "type": "MainObject", "args": ["0"] },
                     "to":   { "type": "Crossroads" } },
  { "type": "Stone", "from": { "type": "Crossroads" },
                     "to":   { "type": "Connection", "args": ["Spawn-A-Side-A1"] } }
]
```

- Road `type`: `"Stone"` or `"Dirt"`.
- Endpoint `type`: `MainObject` (`args: ["<index>"]`), `Connection` (`args: ["<connectionName>"]`),
  `Crossroads` (no args; the zone's crossroads node), or `MandatoryContent`
  (`args` referencing a mandatory-content item).

  > Note: `Crossroads` as a road endpoint is **not** listed in the editor's `KnownValues`
  > (`RoadEndpointTypes` there is only Connection/MainObject/MandatoryContent) but is used widely
  > in official templates — the editor list is incomplete.

### `encounterHolesSettings`

Optional per-zone tuning for encounter "holes" (when `gameRules.encounterHoles` is on):

```jsonc
"encounterHolesSettings": { "affectedEncounters": 0.66, "twoHoleEncounters": 0.66 }
```

- `affectedEncounters` — fraction of encounters that get holes.
- `twoHoleEncounters` — fraction of those that get two holes rather than one.

---

## `connections`

Edges of the zone graph. Each connection joins two zones by name.

```jsonc
{
  "name": "Spawn-A-Side-A1",
  "from": "Spawn-A",
  "to":   "Side-A1",
  "connectionType": "Direct",
  "road": true,
  "guardEscape": false,
  "simTurnSquad": false,
  "guardValue": 3000,
  "guardWeeklyIncrement": 0.10,
  "guardZone": "Side-A1",
  "guardMatchGroup": "hub_guard_A",
  "gatePlacement": "Center",
  "length": 1.0,
  "portalPlacementRulesFrom": [ … ],
  "portalPlacementRulesTo":   [ … ]
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | Optional name; referenced by `roads` endpoints. |
| `from` / `to` | string | Zone names this edge links. |
| `connectionType` | string | See table below. |
| `road` | bool | Whether a visible road is drawn along the connection. |
| `guardEscape` | bool | Whether the border guard can be bypassed/escaped **[meaning unverified — see 06]**. |
| `simTurnSquad` | bool | Treat the guard as a "simulated-turn squad" that reacts to nearby heroes **[editor-inferred]**. |
| `guardValue` | int | Strength of the border guard army. |
| `guardWeeklyIncrement` | double | Compounding weekly growth of the border guard — ×`(1 + increment)` per week (confirmed; see zone field). |
| `guardZone` | string | Which of the two zones "owns"/hosts the guard. |
| `guardMatchGroup` | string | Names a group of connections whose guards are kept **identical** (for mirrored balance, e.g. `"hub_guard_A"`, `"graph_guard_Graph-1"`). |
| `gatePlacement` | string | Where a gate is placed on the connection. Only known value: `"Center"`. |
| `length` | double | Relative length hint for the connection. |
| `portalPlacementRulesFrom` / `…To` | array | Placement rules for the portal endpoints (only for `Portal` connections). Use the same rule grammar as content — see [04](04-content-and-placement.md#placement-rules). In practice: `[{ "type": "Crossroads", "args": [] }]`. |

### Connection types

| `connectionType` | Meaning |
|------------------|---------|
| `Direct` (810) | A guarded, traversable border between adjacent zones (the standard road link). |
| `Default` (382) | Equivalent to a default direct link **[editor-inferred]**; the generator's default connection behaviour. |
| `Portal` (184) | A teleport link between (usually non-adjacent) zones. Entry/exit positions are set by `portalPlacementRules*`. |
| `Proximity` (133) | A non-traversable adjacency hint — tells the generator two zones neighbour each other without creating a road/guard. Used to shape structured layouts. |
| `GladiatorArena` (2) | Special link to a Gladiator Arena zone for arena win conditions. |

---

## Zone layouts

The `layout` string on a zone names a terrain/obstacle/encounter profile. Layouts may be defined
inline in the root `zoneLayouts` array, or be **built-in** to the game (many templates reference
layouts they don't define inline, so the game must ship defaults). Inline layout schema and the
full field list are in [04](04-content-and-placement.md#zone-layout-definitions).

**Layout names used by official templates** (with how often they appear; role is by naming
convention/usage):

| Layout name | Typical role |
|-------------|--------------|
| `zone_layout_spawns` (227) / `zone_layout_spawn` (47) / `zone_layout_player_spawn` / `zone_layout_second_spawn` / `zone_layout_ai_spawn` | Player / spawn zones |
| `zone_layout_center` (352) / `zone_layout_center_zone` | Central / hub-style zones |
| `zone_layout_sides` (162) / `zone_layout_side_zone` / `zone_layout_side_spawn_zone` | Side / neutral border zones |
| `zone_layout_treasure_zone` (50) / `zone_layout_treasures` (41) / `zone_layout_treasure` / `zone_layout_supertreasure_zone` | Treasure zones (rich content) |
| `zone_layout_start_zone` / `zone_layout_back` / `zone_layout_leaf` | Structural roles (start, dead-end, leaf) |
| `zone_layout_wincondition_zone` | Hold-City / objective zone |

> These names are **conventions**, not enforced enums — `zone_layout_center` is by far the most
> common and is used generically. The role mapping above is inferred from where each is used.

---

## Topology shapes

Templates store explicit zones+connections; there is **no topology field**. The recognizable
"shapes" (and the editor's generation modes that can produce them) are:

| Shape | Description | Example templates / previews |
|-------|-------------|------------------------------|
| **Ring / "Default"** | Zones in a circle, each linked to its two neighbours. | *Spider* (8 players ringed around a centre) |
| **Chain** | Zones in a line. | *Highway*, *Hallway*, *Staircase* |
| **Hub & Spoke** | All zones link to one central zone; players never border each other. | *Crossroads*, *Junction* |
| **Cross / Jebus** | Players at the arms of a cross, rich shared centre. | *Jebus Cross*, *Slow Dance Cross* |
| **Random (Delaunay)** | Zones at random positions, connected by spatial adjacency. | *Anarchy* (random variants) |
| **Balanced** | Concentric rings by quality tier (low outer → high inner). | *All Around*, *Clover* |

The preview PNGs visualise these graphs directly — see the legend in
[06's preface](06-open-questions-and-tests.md) / below.

### Reading the preview images

Two visual styles ship in `../Templates`:

- **Official (parchment background):** brown numbered discs = **player spawns** (the number is the
  player slot); small house glyphs = **cities/towns**; small filled dots = **neutral content
  nodes**; thin lines = **connections**. (e.g. *All Around*, *Arcade*, *Spider*, *Jebus Cross*.)
- **Editor-rendered (dark background):** **[editor-inferred legend]** green numbered discs =
  player spawns with a small `🏠N` badge = castle count; bronze/silver/gold discs = neutral zones
  by quality tier (low/medium/high) each showing `🏠N`; gold lines = direct connections; bluish
  lines = portals; a gold-outlined house = the Hold-City objective. (e.g. *Clover*.)

Both depict the same thing: the zone graph and where players start.
