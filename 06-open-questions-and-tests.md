# 06 — Open Questions & Test Plan

This page lists everything that could **not** be confidently resolved from the templates, preview
images, and the community editor, then gives concrete in-game tests to resolve them. Three
ready-to-run probe templates live in [`test-templates/`](test-templates/).

When you run a test, copy the `.rmg.json` into
`…\HeroesOldenEra_Data\StreamingAssets\map_templates\`, start a game on it, and report back what
you observe — I can then turn the answers into confirmed documentation.

---

## A. Open questions

### Guard / value system
1. **`guardValue` units.** *(Partially confirmed.)* `guardValue` is an abstract army-**value**
   budget the generator fills with biome/faction-appropriate creatures — *not* a fixed count. Two
   borders both at `guardValue: 10000` gave 20–49 Grolls (T3) vs. 50–99 Ra'Shoths (upg. T1) + 20–49
   Votaries (upg. T2): same budget, different composition, counts inversely proportional to creature
   tier. **Still open:** the absolute value-per-point scale (what total strength `10000` represents),
   and whether `guardMultiplier`/`guardRandomization` apply before or after creature selection. Run
   **Probe-Guards** (5000 vs 50000) to check the count scales ~10×.
2. **`guardCutoffValue`.** Confirm it's the threshold below which a guard is dropped (content left
   unguarded). What exactly is compared against it — the per-object guard value, or the zone budget?
3. **`guardMultiplier` vs `guardRandomization` vs `guardWeeklyIncrement`.** Confirm multiplier
   scales final guard value, randomization is the ± spread, and weekly increment grows guards over
   time. Is weekly increment compounding or linear?
4. **`guardReactionDistribution` (the 6-int array).** What do the six buckets mean? Hypothesis:
   creature *disposition/aggression* tiers (how likely guards are to fight vs. let you pass, or
   reward-vs-threat banding). Which index is "weakest/most passive" and which is "strongest"?
5. **Content value budgets.** *(Partially confirmed.)* Non-zero `guardedContentValue` produces
   guarded content as expected (a zone on `…_t4_base` with `guardedContentValue: 300000` yielded a
   Dragon Utopia + an assorted mix of guarded buildings). **Still open:** the both-`0` case (as in
   *Symmetry*) — does any guarded content spawn then? — and the precedence between the absolute and
   per-area budgets when both are non-zero.
6. **`valueOverrides`.** *(Confirmed.)* Overrides work, **but `variant` must match the object's
   concrete variant — `-1` is not a wildcard here.** With `variant: -1` the override did nothing;
   re-keyed to the object's concrete `variant: 0` it raised the guard to ~20× the surrounding
   borders. This held for both an ordinary object (`tree_of_abundance`) and a `dragon_utopia`, so a
   utopia's variant-driven guard **can** be overridden — not a special case. (Recorded in
   [04](04-content-and-placement.md#value-overrides).)

### Zones & layout
7. **`size` baseline.** Is `1.0` a fixed fraction of the map, or normalised against the sum of all
   zone sizes? (i.e. does doubling every zone's size change anything?)
8. **`crossroadsPosition`.** *(Partially confirmed.)* `1` does route the zone's roads through a
   convergence point near the zone centre (roads meet by the central town), but there is **no
   distinct visual "crossroads" object/marker** — it renders as one continuous road. `0` = none.
   **Still open:** whether values other than 0/1 occur and what the integer would then encode.
9. **`zone_layout_*` names.** Are these enforced/known to the game, or arbitrary labels that only
   matter if defined inline in `zoneLayouts`? (A template referencing `zone_layout_spawns` without
   defining it implies built-in layouts exist — confirm and ideally enumerate them.)
10. **`diplomacyModifier` range/effect.** Confirm negative = harder neutral recruitment; what's the
    scale (is `-0.5` "half chance" or "−50 percentage points")?
11. **`encounterHolesSettings`.** Confirm `affectedEncounters`/`twoHoleEncounters` are fractions
    and that they only apply when `gameRules.encounterHoles` is true.

### Connections
12. **`Default` vs `Direct`.** Are these mechanically identical? If not, how does `Default` differ?
13. **`Proximity`.** Confirm it creates adjacency without a traversable road/guard.
14. **`guardEscape`, `simTurnSquad`.** Exact effects unverified.
15. **`guardMatchGroup`.** Confirm all connections sharing a group string get identical guards.
16. **`length`.** Does it affect actual on-map distance, or is it only a generation hint?
17. **Portal placement rules.** Confirm portals require `portalPlacementRulesFrom/To` and that
    `Crossroads` placement puts the portal at the zone crossroads.

### Faction / biome selectors
18. **Faction selectors.** *(`differentFrom` confirmed.)* A neutral town with
    `FromList args:["differentFrom: 0 Spawn-A","differentFrom: 0 Spawn-B"]` did get a faction
    different from **both** players' starting factions. **Still open:** the `Match` forms —
    confirm `{"type":"Match","args":["0"]}` = "same faction as main object 0" and that
    `["0","Spawn-B"]` references another zone's object.
19. **Biome catalog completeness.** *(Mapping confirmed.)* `Grass`/`Snow`/`Lava` produced exactly
    grassland/snow/lava zones, and an explicit `FromList` biome overrode faction-native terrain. So
    the name→terrain mapping holds and explicit biomes are honored. **Still open (minor):** whether
    `Dirt, Deathland, Autumn, Sand` are the *complete* set (untested, but they appear in official
    `FromList` args).

### Rules / root
20. **`win_condition_2`.** What is it? (Unused by official templates.)
21. **`heroLighting` / `heroLightingDay`.** What does this win condition do (note the spelling)?
22. **Non-square maps.** Does `sizeX != sizeZ` generate correctly? No official template uses it.
23. **Experimental sizes > 240.** Do sizes up to 512 generate playable maps?
24. **Multiple `variants`.** Confirm exactly one variant is chosen per game (vs. combined), and
    whether selection is uniform-random.
25. **`gameMode` effects.** Does `SingleHero` enforce one hero independent of `heroCount*`, or do
    they need to agree?

---

## B. Tests

### Test 1 — Baseline sanity & guard scale — [`Doc-Probe-Base.rmg.json`](test-templates/Doc-Probe-Base.rmg.json)
A minimal **2-player hub**: `Spawn-A` (Player1) and `Spawn-B` (Player2) each connect to a neutral
`Center` city. Uses only generic, game-bundled content pools (`classic_template_pool_random_t2/t4`,
`content_pool_general_resources_*`) so it should load if those pools exist in your build.

**Observe & report:**
- Does the map generate and start at all? (Validates the whole minimal skeleton — answers many
  structural unknowns at once.)
- Roughly how strong are the border-guard armies on the two `Direct` connections (both set to
  `guardValue: 10000`)? Note the creatures/quantities — this calibrates **Q1**.
- Does guarded content (a Dragon Utopia + buildings) actually appear in `Center`, given the
  budgets set? (**Q5**)
- Is there a visible crossroads / road junction in `Center` (`crossroadsPosition: 1`)? (**Q8**)

**✓ Confirmed by this probe:** the minimal skeleton generates and starts; both players spawn (in
opposite bottom corners) with a starting town; mandatory mines (wood/ore/gold) and dwellings
(low+high tier) appear in spawns; `Direct` connections are traversable end-to-end (you can reach
the opponent through `Center`); roads render town→connection→centre; the neutral town's faction
differs from both players. **Note:** zone *names* (`Spawn-A`, etc.) are internal only — there's no
in-game way to tell which authored zone became which on-map position; actual placement is decided
by `orientation`/the generator.

### Test 2 — Biome mapping — [`Doc-Probe-Biomes.rmg.json`](test-templates/Doc-Probe-Biomes.rmg.json)
Same map, but each zone is forced to a fixed biome: `Spawn-A → Grass`, `Spawn-B → Snow`,
`Center → Lava` (via `zoneBiome`/`contentBiome` = `{"type":"FromList","args":["<Biome>"]}`).

**Observe & report:** the terrain of each of the three zones. This confirms the **biome-name →
terrain** mapping (**Q19**) and that explicit `FromList` biomes are honoured.

**✓ Confirmed:** Grass/Snow/Lava zones rendered as expected, overriding faction-native terrain.

### Test 3 — Guard scaling & value override — [`Doc-Probe-Guards.rmg.json`](test-templates/Doc-Probe-Guards.rmg.json)
Same map, but the two border connections are deliberately **asymmetric**: `Spawn-A→Center` guard
`5000` vs `Spawn-B→Center` guard `50000` (10×). A `valueOverride` also forces every
`dragon_utopia` to `guardValue: 200000`.

**Observe & report:**
- Compare the two border armies. Is the 50000 border visibly ~10× the 5000 border? Record the
  actual stacks for both — this gives the **`guardValue` → army** mapping (**Q1**).
- Is the Dragon Utopia in `Center` far more heavily guarded than its normal version? (**Q6**)

### Test 4 — value-override mechanism — [`Doc-Probe-Guards-v2.rmg.json`](test-templates/Doc-Probe-Guards-v2.rmg.json)
Follow-up to Test 3's null result. Both borders are equal (`10000`) so the override is the only
variable. `Center` now gets two **fixed-variant-0** mandatory objects — a `tree_of_abundance`
(guarded; exactly how *Chosen One* uses it) and a `dragon_utopia` — and `valueOverrides` raises
**both** to `guardValue: 200000`, keyed to the **same concrete variant 0** (no `-1`).

**Observe & report:**
- Is the **Tree of Abundance** now guarded by a huge stack (far bigger than the 10000 borders)?
  If yes → `valueOverrides` works when the variant matches, and `-1` was the problem in Test 3 (**Q6**).
- Is the **Dragon Utopia** now hugely guarded too? If the Tree balloons but the Utopia doesn't →
  Utopia's guard is variant-driven and can't be value-overridden (a distinct finding). If both
  balloon → overrides apply to utopias as well.

**✓ Confirmed by this probe:** both objects' guards ballooned to ~20× the borders (two 20–49 T7
stacks each), so overrides work with a matching concrete variant, and Dragon Utopia is overridable
like any object. Resolves Q6.

### Test 5 — guard reaction distribution — [`Doc-Probe-Reaction.rmg.json`](test-templates/Doc-Probe-Reaction.rmg.json)
Resolves **Q4**. A 2-player **chain** `Spawn-A — Neutral-React0 — Neutral-React5 — Spawn-B`. The two
neutral zones are **identical** (same layout, pools, content values `250000` guarded, the same
guaranteed guarded resource/unit banks, `guardMultiplier: 1.0`, `guardRandomization: 0`) except for
`guardReactionDistribution`: `Neutral-React0` = `[100,0,0,0,0,0]` (all bucket 0), `Neutral-React5` =
`[0,0,0,0,0,100]` (all bucket 5).

Identify the two neutral zones by adjacency: **React0 is the neutral next to Player 1**, **React5 is
the neutral next to Player 2** (you can read player slots even though zone names aren't shown).

**Observe & report — compare the guards in the two neutral zones:**
- Hover/inspect the guard stacks on the guarded banks in each zone. Does the game show a
  **disposition / reaction** (e.g. something like savage / hostile / aggressive / … / "will join")?
  What does React0 show vs React5? This reveals what the six buckets index (likely creature
  disposition/aggression, lowest→highest, or willingness to join/flee).
- Are the guard **sizes/strengths roughly the same** in both zones (they should be — only the
  reaction varies, not the value)? If sizes differ too, the distribution affects strength as well.
- Anything else that differs between the two zones' guards (behaviour when approached, join offers,
  flee chance).

If the two extremes look identical in-game, try intermediate buckets (e.g. `[0,0,100,0,0,0]`) to
find which index changes what.

### Further tests you can author by editing the probes
- **Q4 (reaction distribution):** clone Probe-Base, give `Center` `guardReactionDistribution`
  `[100,0,0,0,0,0]` and a copy with `[0,0,0,0,0,100]`; compare guard behaviour/strength between
  the two extremes.
- **Q7 (`size`):** make one spawn `size: 0.5` and the other `size: 2.0`; compare zone areas.
- **Q12/Q13 (`Default`/`Proximity`):** change one connection's `connectionType` to `Default` and
  another map's to `Proximity`; check whether you can walk between the zones.
- **Q20/Q21 (win conditions):** set `displayWinCondition` to `win_condition_2` and try each
  `winConditions` flag (`heroLighting`, etc.) one at a time; report the in-game objective text.
- **Q22/Q23 (sizes):** set `sizeX: 96, sizeZ: 160` (non-square) and `sizeX:sizeZ: 320`
  (experimental); report whether generation succeeds.

> ⚠️ These probe templates are **unverified** — they're built to match the structure and pool
> references of official templates, but no one has confirmed they generate in your game build yet.
> If `Doc-Probe-Base` fails to load, the most likely cause is a content-pool ID that doesn't exist
> in your version; tell me the error and I'll swap in pool IDs from a template you know works (or
> inline a minimal pool once we confirm the inline `contentPools` schema).
