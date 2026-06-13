# 04 — Content & Placement System

This page covers how objects (mines, dwellings, banks, pickups, artifacts, Pandora boxes, …) are
chosen and placed in zones. Five mechanisms cooperate:

1. **Content pools** — weighted bags a zone draws random content from (referenced by ID).
2. **Content lists** — reusable named groups of objects that pools `includeLists`.
3. **Mandatory content** — objects guaranteed to spawn in a zone.
4. **Content count limits** — caps on how many of a given object may appear.
5. **Placement rules** — constraints on *where* an object spawns.

> **Where definitions live.** Official templates leave the root `contentPools` and `contentLists`
> arrays **empty** and reference pools/lists *by ID*. Those IDs resolve to definitions shipped with
> the game. The community editor bundles copies of this game data under
> `../Olden-Era---Template-Generator/Olden Era - Template Editor/GameData/GeneratorData/`, which is
> the best available description of the pool/list shapes documented below.

---

## Content pools

A zone references pools in `guardedContentPool`, `unguardedContentPool`, `resourcesContentPool`
(see [03](03-variants-zones-connections.md#content-budgets--pools)). A pool definition:

```jsonc
{
  "name": "content_pool_default_guarded",
  "valueDistribution": {                 // optional: bias selection by object price/value tier
    "priceBounds": [3999, 6999, 12999, 15999],
    "weights":     [6, 8, 10, 6, 0]      // one more weight than bounds (the buckets between/around bounds)
  },
  "groups": [
    {
      "weight": 1,                        // relative weight of this group within the pool
      "includeLists": [ "content_list_basic_buildings", "content_list_building_random_hires", … ],
      "content": [                        // optional inline entries (sid + weight [+ variant])
        { "sid": "random_item_common", "weight": 100 }
      ]
    }
  ],
  "bans": [ { "sid": "stables" } ]        // optional: objects excluded from this pool
}
```

| Field | Meaning |
|-------|---------|
| `name` | Pool ID referenced by zones. |
| `valueDistribution.priceBounds` | Ascending value thresholds defining price buckets. |
| `valueDistribution.weights` | Weight per bucket (controls how much cheap vs. expensive content appears). |
| `groups[]` | Each group is a weighted sub-bag combining `includeLists` and inline `content`. |
| `groups[].weight` | Relative chance of drawing from this group. |
| `groups[].includeLists` | Content-list IDs to pull from. |
| `groups[].content` | Inline `{ sid, weight, variant? }` entries. |
| `bans` | `{ sid }` entries removed from the pool's candidate set. |

### Pool naming conventions

| Pattern | Meaning |
|---------|---------|
| `content_pool_default_{guarded,unguarded,resources}` | Generic defaults. |
| `content_pool_general_resources_{start,side}_zone_{very_poor,poor,medium,rich}` | Resource pools by zone role + richness. |
| `classic_template_pool_random_t{0..5}_<variant>` and `classic_template_pool_random_unguarded_t{0..5}_<variant>` | **Tiered random pools** (t0 = weakest content, t5 = strongest; tier = value band set by `valueDistribution.priceBounds`). Files live in `content_pools/random_pools/classic_version/`. |
| `content_pool_template_<template>_<role>` | Template-specific custom pools. |

**Tiered-pool variants.** Each tier `t0..t5` comes in several `<variant>` flavours. `_base` is the
broad **mixed** pool — it `includeLists` random-item pickups, pandora/scroll boxes, storage, hero
exp/magic/buff/stat-skill buildings, interact buildings, resource banks (guarded + unguarded),
**dwellings** (`random_hires`), and unit banks, each weighted. The specialized siblings bias the
content toward one category: `_item`, `_pandora`, `_hire` (dwellings), `_unit_bank`, `_res_bank`,
`_stat`, `_magic`. *(Confirmed in-game: a zone on `…_t4_base` yields an assorted mix of guarded
buildings — banks, stat/magic buildings, dwellings, etc.)*

> **Terminology:** the in-game term for the "hire buildings" produced by `random_hire_*` /
> `content_list_building_random_hires*` is **dwelling**.

---

## Content lists

A content list is a reusable, weighted, optionally biome-filtered set of objects, pulled in by a
pool's `includeLists`.

```jsonc
{
  "name": "basic_content_list_rare_resources",
  "content": [
    { "sid": "resource_crystals",  "weight": 50 },
    { "sid": "resource_gemstones", "weight": 100, "biome": "Grass" },  // biome-specific override
    { "sid": "resource_mercury",   "weight": 100, "biome": "Snow"  }
  ]
}
```

| Entry field | Meaning |
|-------------|---------|
| `sid` | Object SID (see [05](05-id-reference.md)). |
| `weight` | Selection weight relative to siblings. |
| `variant` | Optional variant index (see [content variants](#content-variants)). |
| `biome` | Optional biome filter; entry only eligible in that biome. |

### List naming conventions

`basic_content_list_*` (in `basic_content_lists.json`) and `content_list_*` (in
`generator_content_lists.json`) for generic lists; `template_pool_<template>_*` /
`custom_content_lists_<template>.json` for template-specific ones. Names are descriptive of what
they contain, e.g. `content_list_building_random_hires_high_tier`,
`basic_content_list_building_guarded_resource_banks_tier_2`,
`content_list_pickup_pandora_box_army_high_tier`. A representative catalog of list IDs is in
[05 — Content lists](05-id-reference.md#content-lists-include-lists).

---

## Mandatory content

Root `mandatoryContent` is an array of **named groups**. A zone opts into a group by listing its
name in the zone's `mandatoryContent` array; every item in the group is then guaranteed to spawn
in that zone.

```jsonc
{
  "name": "mandatory_content_spawn",
  "content": [
    { "sid": "mine_wood", "isMine": true, "isGuarded": false,
      "rules": [ { "type": "MainObject", "args": ["0"], "targetMin": 0.15, "targetMax": 0.35, "weight": 1 },
                 { "type": "Crossroads", "args": [],   "targetMin": 0.15, "targetMax": 0.30, "weight": 1 } ] },
    { "sid": "dragon_utopia", "variant": -1 },
    { "includeLists": [ "content_list_building_random_hires_high_tier" ] }
  ]
}
```

Each content item:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | Optional item name (for road targeting via `MandatoryContent` endpoints). |
| `sid` | string | Object to place. |
| `variant` | int | Variant index; `-1` means "random/any variant" (see below). |
| `isGuarded` | bool | Force the item guarded (`true`) or unguarded (`false`). |
| `isMine` | bool | Treat as a resource mine. |
| `soloEncounter` | bool | Place alone, not clustered with other content. |
| `includeLists` | string[] | Instead of a fixed `sid`, pull one object from these content lists. (Repeating an item with the same `includeLists` requests *that many* objects — e.g. listing `content_list_building_random_hires_low_tier` six times guarantees six low-tier hire buildings.) |
| `rules` | array | Placement constraints — see [placement rules](#placement-rules). |

---

## Content count limits

Root `contentCountLimits` is an array of named cap-sets. A zone opts in via its
`contentCountLimits` array. Each cap-set:

```jsonc
{
  "name": "content_limits_spawn",
  "playerMin": null,        // optional: only apply within this player-count range
  "playerMax": null,
  "limits": [
    { "sid": "stables",        "maxCount": 1 },
    { "sid": "crystal_trail",  "variant": -1, "maxCount": 1 }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `playerMin` / `playerMax` | Optional player-count gating for the whole cap-set. |
| `limits[].sid` | Object SID being capped. |
| `limits[].variant` | Optional variant the cap applies to (`-1` = any). |
| `limits[].maxCount` | Maximum allowed in the zone. |

---

## Placement rules

Rules constrain where an object (or portal endpoint) may be placed. They appear in a content
item's `rules` array and in connection `portalPlacementRules*`.

```jsonc
{ "type": "MainObject", "args": ["0"], "targetMin": 0.2, "targetMax": 0.4, "weight": 1 }
```

| Field | Meaning |
|-------|---------|
| `type` | Reference frame for the distance constraint (see below). |
| `args` | Frame-specific arguments. |
| `targetMin` / `targetMax` | Target distance band, as a **normalised fraction** (0 = at the reference, 1 = far). The object is steered toward `[min, max]`. |
| `weight` | Relative importance when several rules combine. `weight: 0` appears in some data; treat as "advisory/disabled" **[editor-inferred]**. |

**Rule types actually used in official templates** (with frequency):

| `type` | `args` | Constrains distance to… |
|--------|--------|--------------------------|
| `MainObject` (2594) | `["<index>"]` — index of a main object (e.g. `"0"` = the zone's spawn/city) | the chosen main object. |
| `Road` (1053) | `[]` | the nearest road. |
| `Crossroads` (1032) | `[]` | the zone's crossroads node. |

The editor additionally exposes *Guarded*, *Variant*, and *SoloEncounter* "rules" in its UI, but
those compile down to the `isGuarded` / `variant` / `soloEncounter` **fields** on the content item
(not to entries in the `rules` array). Distance bands map to editor presets roughly as:
Next-To ≈ 0.05–0.1, Near ≈ 0.1–0.25, Medium ≈ 0.25–0.5, Far ≈ 0.5–0.75, Very-Far ≈ 0.75–0.9.

---

## Content variants

The `variant` integer on a content item / count-limit / value-override selects a sub-type of an
object. `variant: -1` means "any/random". Concrete variant tables **[from the editor's
`VariantMapping`]**:

**`dragon_utopia`** — guard tier: `0` Small, `1` Medium, `2` Large, `3` Maximum.

**`monty_hall`** — reward rarity: `0` Common, `1` Rare, `2` Epic, `3` Legendary.

**`pandora_box`** — reward type & tier:

| Variant | Reward | | Variant | Reward |
|--:|---|---|--:|---|
| 0–3 | Gold T1→T4 | | 15–18 | All-Stats T1→T4 |
| 4–7 | Experience T1→T4 | | 19–22 | Spells: Daylight / Nightshade / Arcane / Primal |
| 8–14 | Units T1→T7 | | 23–27 | Spells T1→T5 |

For SIDs without a variant table, `variant` is usually omitted or `-1`.

---

## Value overrides

Root `valueOverrides` globally overrides the **guard value** of a specific object SID+variant,
wherever it spawns in the map:

```jsonc
"valueOverrides": [
  { "sid": "tree_of_abundance", "variant": 0, "guardValue": 100000 }
]
```

Use this to make a particular object much more (or less) heavily guarded than its pool default.

> **`variant` must match the object's concrete variant — `-1` is *not* a wildcard here (confirmed).**
> Unlike `mandatoryContent`/`contentCountLimits` (where `-1` = any/random), an override only applies
> when its `variant` equals the variant the object actually spawned as. Every official template that
> uses `valueOverrides` specifies a concrete variant (e.g. *Chosen One*: `tree_of_abundance,
> variant: 0`).
>
> Test sequence: an override with `variant: -1` had **no effect** (the object's real variant didn't
> match `-1`); the same override re-keyed to the object's **concrete variant 0** raised the guard to
> ~20× the surrounding borders. This worked identically for an ordinary object (`tree_of_abundance`)
> *and* for `dragon_utopia` — so a utopia's variant-driven guard **can** be value-overridden; it is
> not a special case.

---

## Zone layout definitions

The root `zoneLayouts` array defines named terrain/obstacle/encounter profiles referenced by a
zone's `layout` field. Schema:

```jsonc
{
  "name": "zone_layout_sides",
  "obstaclesFill": 0.60,        // obstacle density (0–1) over normal terrain
  "obstaclesFillVoid": 0.60,    // obstacle density over "void"/unused terrain
  "lakesFill": 0.30,            // water density (0–1)
  "minLakeArea": 15,            // smallest lake size in tiles
  "elevationClusterScale": 0.256,
  "elevationModes": [           // weighted elevation profiles
    { "weight": 1, "minElevatedFraction": 0.0, "maxElevatedFraction": 0.0 }
  ],
  "roadClusterArea": 128,       // target area for road clustering
  "guardedEncounterResourceFractions": {   // split of guarded encounters that are resources
    "countBounds": [],
    "fractions": [ 0.5 ]
  },
  "ambientPickupDistribution": {           // scattered pickup behaviour
    "repulsion": 1.0,         // min spacing between pickups
    "noise": 0.3,             // randomness
    "roadAttraction": 0.5,    // pull toward roads
    "obstacleAttraction": 0.0,// pull toward obstacles
    "groupSizeWeights": [ 4, 1, 1 ]   // weights for pickup cluster sizes (1,2,3,…)
  }
}
```

> The editor's bundled `default_zone_layouts.json` additionally shows fields like
> `guardedEncounterDensity`, `guardedEncounterSizeDistribution`,
> `unguardedEncounterDensity`/`…SizeDistribution`, and `ambientPickupDensity`. Not all of these
> appear in every official template's inline layouts; the shape above matches the official
> *Symmetry* file. Treat extra fields as optional tuning.

### Encounter templates (background)

Guarded/unguarded content is physically laid out using small pre-authored **encounter
footprints** (e.g. `rmg_2building_1_1_encounter_3x3_1`). These are game-internal building/guard/
pickup micro-layouts (3×3 … 5×5 tiles), not something templates author directly — templates only
influence *which* and *how dense* via zone-layout fields. Filename grammar:
`rmg_<#building>building_<config>_[pickups_]encounter_<WxH>_<n>`.
