# Heroes: Olden Era — `.rmg.json` Map Template Documentation

This documentation describes the **random map generation (RMG) template format** used by
*Heroes of Might and Magic: Olden Era*. These `.rmg.json` files live in the game's
`HeroesOldenEra_Data\StreamingAssets\map_templates` folder and are read by the game's
map generator to build a playable map each time a game is started from a template.

It was produced by reverse-engineering the 61 official templates in `../Templates`, their
preview images, and the community **[Olden Era Template Generator]** (a C# editor whose model
classes and bundled game data are the best available description of the schema). Where a fact
comes only from the editor (and is therefore the editor author's interpretation rather than
confirmed game behaviour) it is flagged **[editor-inferred]**. Genuine unknowns and ways to
test them in-game are collected in **[06 — Open Questions & Test Plan](06-open-questions-and-tests.md)**.

[Olden Era Template Generator]: ../Olden-Era---Template-Generator

---

## How map generation works (mental model)

A template is **not** a map. It is a *recipe*. It describes:

1. **A grid** (`sizeX` × `sizeZ`) the map is built on.
2. **One or more `variants`** — alternative whole-map layouts. The generator picks one variant
   per game. Each variant contains a set of **zones** and the **connections** between them.
3. **Zones** — regions of the map. Each zone has a *role* (player spawn, neutral/side, treasure,
   centre, …), a physical `size`, terrain/obstacle settings (its `layout`), and references to
   **content pools** that decide which objects spawn inside it and how strongly they are guarded.
4. **Connections** — the graph edges between zones (roads, portals, guarded borders).
5. **Content definitions** — `mandatoryContent`, `contentCountLimits`, and (by reference)
   content pools/lists that populate zones with mines, dwellings, banks, pickups, etc.
6. **Game rules** — hero counts, win conditions, banned items/spells, starting bonuses.

The generator then places zones according to the connection graph and per-variant
`orientation`, fills each zone with content drawn from its pools (subject to per-area *value*
budgets and count limits), wires up guards, roads and portals, and emits a concrete map.

### Important: "topology" is a generator concept, not a template field

The community editor talks about *topologies* (Ring, Chain, Hub, Random, Balanced…). **Templates
do not store a topology field.** A template just contains an explicit list of zones and
connections; the "topology" is simply the *shape* of that hand-authored graph. The editor's
topology modes are strategies for *producing* such graphs. See
[03 — Variants, Zones & Connections](03-variants-zones-connections.md#topology-shapes).

---

## Document index

| File | Contents |
|------|----------|
| [01 — Root & File Structure](01-root-and-structure.md) | Top-level fields, file conventions, a fully annotated minimal example. |
| [02 — Game Rules & Win Conditions](02-game-rules-and-win-conditions.md) | `gameRules`, hero counts, `winConditions`, `globalBans`, start bonuses, `displayWinCondition`. |
| [03 — Variants, Zones & Connections](03-variants-zones-connections.md) | `variants`/`orientation`/`border`, the full `zone` field reference, `connections`, `roads`, biomes, topology shapes. |
| [04 — Content & Placement System](04-content-and-placement.md) | Content pools, content lists, `mandatoryContent`, `contentCountLimits`, placement `rules`, `variant` indices, value budgets. |
| [05 — ID Reference](05-id-reference.md) | Catalogs: object SIDs → names, zone layouts, construction SIDs, biomes, spells, bannable artifacts, win-condition IDs, etc. |
| [06 — Open Questions & Test Plan](06-open-questions-and-tests.md) | What we could **not** determine, and concrete in-game tests (with ready-to-run probe templates) to resolve them. |
| [test-templates/](test-templates/) | Minimal templates you can drop into the game to answer specific open questions. |

---

## Quick reference: corpus statistics

Aggregated over all 61 official templates (useful for knowing what's "normal"):

- **`gameMode`**: `Classic` (41) and `SingleHero` (20).
- **`displayWinCondition`**: `win_condition_1` Standard (29), `win_condition_3` Lost-City (18),
  `win_condition_4` (4), `win_condition_5` Hold-City (5), `win_condition_6` Tournament (5).
- **`connectionType`**: `Direct` (810), `Default` (382), `Portal` (184), `Proximity` (133), `GladiatorArena` (2).
- **Orientation `mode`**: `MinimalBoundingSquare` (44) ≫ `BoundingCircle` (12).
- **`mainObjects` types**: `City` (668), `Spawn` (246), `AbandonedOutpost` (40), `GladiatorArena` (1).
- **`placement`**: `Uniform` (709), `Center` (177), `Connection` (61), `NearZone` (2).
- **Placement `rules` types in use**: `MainObject` (2594), `Road` (1053), `Crossroads` (1032).

> Where this documentation gives exact colours, formulas, or generation heuristics drawn from the
> editor source, treat them as a *guide to intent*, not a guarantee of the shipping game's behaviour.
> The authoritative artifacts are the official templates themselves.
