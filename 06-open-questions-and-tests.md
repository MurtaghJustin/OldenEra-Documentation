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
2. **`guardCutoffValue`.** *(Resolved.)* It's the **minimum guard value to keep** — each content
   guard whose value is below it is dropped and the object left unguarded. A cutoff above the zone's
   content guard values removes *all* guards (a cutoff-30000 zone full of cheap guards had **no guards
   at all**). The common `1500` strips trivial sub-1500 guards.
3. **`guardMultiplier` vs `guardRandomization` vs `guardWeeklyIncrement`.** *(Mostly resolved.)*
   `guardMultiplier` scales the zone's **content** guard values (a ×2.0 zone ≫ an otherwise-identical
   ×0.5 zone) and does **not** affect border/connection guards. `guardWeeklyIncrement` is
   **compounding** — ×`(1 + increment)` per week (`1.0` doubled the guard each week: 1×→2×→4×).
   `guardRandomization` is a **per-guard ± spread** on guard values (confirmed: `0.25` gave clearly
   varied guard sizes across six identical objects vs. uniform at `0.0`). So the whole trio is
   resolved. (Keep `guardRandomization` ≤ `0.25` — the official max; a `0.5` test broke content
   placement.)
4. **`guardReactionDistribution` (the 6-int array).** *(Largely resolved.)* Six weights assigning
   each of a zone's **content guards** a friendliness/disposition tier (index `0` = least friendly →
   `5` = friendliest); it does not set strength, and **border guards ignore it (always Fight)**. The
   game shows only **Will Fight / Will Flee / Will Join** (via a reveal spell, vs the viewing hero's
   army). Resolution rules, confirmed by testing:
   - More hero overpower → shifts from Fight toward Flee (no Diplomacy) / Join (with Diplomacy).
   - **Tiers 0–4 require Diplomacy to Join** — *without* Diplomacy they never join, even with an
     absurdly large army (only Fight/Flee). A higher tier joins at a smaller army advantage.
   - **Tier 5 joins even without Diplomacy** ("free-join").
   - "Mixed" reactions within one zone = per-object guard-size differences vs the hero.
   **Minor remainder:** the precise army-advantage threshold separating tiers 1–4 from each other
   (they behave similarly — all Diplomacy-gated — differing only in how easily they flip to Join).
5. **Content value budgets.** *(Partially confirmed.)* Non-zero `guardedContentValue` produces
   guarded content as expected (a zone on `…_t4_base` with `guardedContentValue: 300000` yielded a
   Dragon Utopia + an assorted mix of guarded buildings). **Open:** the both-`0` case (as in
   *Symmetry*) — does any guarded content spawn then? — and the precedence between the absolute and
   per-area budgets when both are non-zero. **Mostly resolved:** both-zero (`value: 0` +
   `perArea: 0`, pool referenced) produced **no guarded content** — only mandatory/explicit objects
   (how *Symmetry* runs on all-zero values). Each budget **alone** produces content: absolute works,
   and **`guardedContentValuePerArea` alone also works, scaling with zone area** (a `perArea: 2000`
   zone yielded *more* than a `value: 150000` zone). **Still open (minor):** how absolute + per-area
   combine when both are non-zero (add vs. override). *(Note: a suspected
   "guarded mandatory content needs budget" effect was **not** confirmed — raising the budget back up
   did not restore dropped guarded mandatory objects, so that disappearance had another cause; see
   Test 12 notes.)*
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
20. **`win_condition_2`.** *(Resolved.)* = **"Capital Capture"**, and it **changes the win
    mechanic** — you must capture the enemy capital to win (SingleHero: killing the enemy's only hero
    also wins). So `displayWinCondition` is not label-only; it selects the actual headline victory
    condition. Confirmed by isolation (Test 8): `win_condition_1` + `heroLighting` gave plain Standard
    play with no capital requirement, so the mechanic is `win_condition_2`'s. The game has **no
    separate objectives panel**; the condition shows as a name.
21. **`heroLighting` / `heroLightingDay`.** *(Largely settled.)* It's a **near-universal baseline
    flag** — `true` with `heroLightingDay: 1` in **every** official template (Classic and SingleHero).
    Isolating it (Test 8: `win_condition_1` + `heroLighting` only) produced **no observable effect**
    vs. a map without it — no events, reveals, or rule changes. So it is almost certainly a baseline
    rule that's effectively always-on (plausibly the standard "lose with no hero/town" elimination,
    `heroLightingDay` being a grace/check day), not a special mechanic. **Minor remainder:** its exact
    function — could be probed by a long game where a player is reduced to no hero/town with vs.
    without it, but low priority.
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

**✓ Confirmed by this probe:** index `0` = least-aggressive disposition (guards **fled** vs. an
overwhelming hero but **fought** a smaller army), index `5` = friendliest (**offered to join**, even
without Diplomacy); guard sizes stayed similar (disposition ≠ strength) and border guards always
Fight. Middle indices 1–4 still to be mapped (Test 6).

### Test 6 — reaction gradient (middle buckets) — [`Doc-Probe-Reaction-Gradient.rmg.json`](test-templates/Doc-Probe-Reaction-Gradient.rmg.json)
Finishes **Q4**. A 2-player **chain** `Spawn-A — Idx1 — Idx2 — Idx3 — Idx4 — Spawn-B`, where the four
neutral zones are identical except each uses a **single-bucket** `guardReactionDistribution`:
`Idx1 = [0,1,0,0,0,0]`, `Idx2 = [0,0,1,0,0,0]`, `Idx3 = [0,0,0,1,0,0]`, `Idx4 = [0,0,0,0,1,0]`.

**Walking from Player 1 toward Player 2, the neutral zones are in index order 1 → 2 → 3 → 4.**

The game only ever shows **three** reaction values — **Will Fight / Will Flee / Will Join** —
viewable via a reveal spell and computed for the **viewing hero's army**. So the six buckets are
disposition *tiers* mapping onto those three outcomes through an army-strength threshold, not six
distinct labels. The probe **grants a vision spell free to all heroes** (best-guess: Second Sight)
so the value is always viewable; if your game's reveal spell differs, cast that instead.

**Observe & report:** with the **same hero/army**, view each of the four zones' object guards and
note Fight / Flee / Join. Then **repeat with a much stronger (or weaker) army** — the tiers differ
by *where* their Fight↔Flee↔Join threshold sits, so one army strength only shows a slice. Combined
with the endpoints (0 = least aggressive, 5 = friendliest/join) this orders the 1–4 tiers. Also note
whether **Diplomacy** changes the higher buckets.

**✓ Confirmed by this probe:** tiers 1–4 **require Diplomacy to Join** — a non-Diplomacy hero gets
no joins anywhere on the map even with an absurd army (only Fight/Flee); with Diplomacy, the more
the hero overpowers a guard the more it leans Join over Flee (at Idx2, a big enough Diplomacy army
made everything join). Only tier 5 joins without Diplomacy. The fine threshold ordering among 1–4
was not separately resolved.

### Test 7 — win-condition labels & effects — [`Doc-Probe-WinCon.rmg.json`](test-templates/Doc-Probe-WinCon.rmg.json)
Probes **Q20/Q21**. Based on Probe-Base but with `displayWinCondition: "win_condition_2"` and
`winConditions.heroLighting: true` (`heroLightingDay: 1`).

**Observe & report:**
- In the **template picker**, what label/icon does `win_condition_2` show? (**Q20**)
- **In-game**, what objective/victory text does enabling `heroLighting` produce — what does the
  condition actually do, and note the spelling the UI uses (the data says "heroLighting")? (**Q21**)
- To map the rest, edit `displayWinCondition` through the other IDs (`win_condition_1/3/4/5/6`) and
  note each picker label, and toggle other `winConditions` flags one at a time.

### Test 8 — isolate `heroLighting` — [`Doc-Probe-HeroLighting.rmg.json`](test-templates/Doc-Probe-HeroLighting.rmg.json)
Disambiguates Q20/Q21. Identical to Probe-Base but with `displayWinCondition: "win_condition_1"`
(Standard headline — **not** Capital Capture) and `winConditions.heroLighting: true`
(`heroLightingDay: 1`). `heroLighting` is now the only special flag.

**Observe & report:**
- Is the win condition **no longer "Capital Capture"** / no longer requiring you to take the enemy
  capital? If so, that mechanic belonged to `win_condition_2` (confirms Q20 cleanly).
- What does **`heroLighting` actually do** in play — any day-1 message, a hero-related win/loss
  rule, or no observable effect? (**Q21**)

**✓ Confirmed by this probe:** win condition was plain **Standard** (take all enemy towns + kill all
heroes), **no** capital requirement → Capital Capture is `win_condition_2`'s mechanic (closes Q20).
`heroLighting` showed **no observable effect** — and it's set in every official template, so it reads
as an always-on baseline flag rather than a special condition (Q21).

### Test 9 — `guardMultiplier` — [`Doc-Probe-GuardMultiplier.rmg.json`](test-templates/Doc-Probe-GuardMultiplier.rmg.json)
Chain `Spawn-A — Neutral-A — Neutral-B — Spawn-B`. The two neutral zones are identical
(`guardedContentValue: 200000`, same pool, `guardRandomization: 0`) except **`guardMultiplier`**:
`Neutral-A = 0.5`, `Neutral-B = 2.0` (a 4× ratio). Neutral-A is next to Player 1, Neutral-B next to
Player 2.

**Observe & report:** compare the **in-zone content guards** between the two zones — are
Neutral-B's guards roughly **4× stronger** than Neutral-A's (confirming `guardMultiplier` linearly
scales zone guard values)? Also note whether the **border guards** differ (they shouldn't —
`guardMultiplier` is a zone field; borders were left equal) so we learn whether it touches
connection guards. (**Q3**)

**✓ Confirmed:** the ×2.0 zone's content guards were far stronger than the ×0.5 zone's, and the
equal border guards stayed equal — `guardMultiplier` scales content guards only, not connection guards.

### Test 10 — `guardCutoffValue` — [`Doc-Probe-GuardCutoff.rmg.json`](test-templates/Doc-Probe-GuardCutoff.rmg.json)
Same chain; both neutral zones use a **cheap pool** (`…_t1_base`) so they fill with many small
guarded objects. Only **`guardCutoffValue`** differs: `Neutral-A = 0` (keep every guard),
`Neutral-B = 30000` (hypothesis: drop any guard whose value falls below the cutoff).

**Observe & report:** does **Neutral-B have noticeably more *unguarded* / freely-walkable objects**
than Neutral-A (i.e. small guards removed)? Or does the cutoff do something else (e.g. remove the
*objects* rather than just their guards, or merge small guards into bigger ones)? Roughly how many
guarded vs unguarded objects in each zone? (**Q2**)

**✓ Confirmed:** the cutoff-30000 zone (cheap `t1` guards, all below the threshold) ended up with
**no guards at all** — confirming `guardCutoffValue` drops every guard below its value, leaving those
objects unguarded.

### Test 11 — `guardWeeklyIncrement` — [`Doc-Probe-GuardWeekly.rmg.json`](test-templates/Doc-Probe-GuardWeekly.rmg.json)
Same chain. `Neutral-A` (and its border) have `guardWeeklyIncrement: 0.0`; `Neutral-B` (and its
border) have `1.0` (= +100% per week). Both borders start at `guardValue: 20000`.

**Observe & report:** record the **border guard size** (and an in-zone guard) for **Neutral-A vs
Neutral-B** on **day 1, day ~8 (start of week 2), and day ~15 (week 3)**. Neutral-A is the control
(no growth). For Neutral-B, the *pattern* tells us compounding vs linear:
- **Compounding** (×2 each week): week1 → week2 → week3 ≈ 1× → 2× → 4×.
- **Linear** (+100% of base each week): ≈ 1× → 2× → 3×.
Note which it looks like (count bands are fine), and whether the increment hits week boundaries or
accrues daily. (**Q3**)

**✓ Confirmed:** the guard **doubled each week** (1× → 2× → 4×) with `guardWeeklyIncrement: 1.0` —
so it's **compounding**: ×`(1 + increment)` per week.

### Test 12 — `guardRandomization` — [`Doc-Probe-GuardRandomization.rmg.json`](test-templates/Doc-Probe-GuardRandomization.rmg.json)
Same chain. Both neutral zones are identical (`guardMultiplier: 1.0`, larger `size`, low pool value
so they place cleanly) and each gets **six copies of the same guarded object** (`tree_of_abundance`,
variant 0). Only **`guardRandomization`** differs: `Neutral-A = 0.0`, `Neutral-B = 0.25` (±25%, the
maximum value any official template uses).

**Observe & report:** view the guard on each of the six identical objects in both zones.
- In **Neutral-A** (0.0) the six guards should all be **the same size**.
- In **Neutral-B** (0.25) do their sizes **vary** (a spread of roughly ±25%)? Note smallest vs largest.
This confirms `guardRandomization` is a per-guard ± spread. (**Q3**)

> **Iteration notes:** getting the six objects to place reliably was finicky. They appeared in the
> original config (zone `size: 1.0`, `guardedContentValue: 200000`, `guardRandomization: 0.0`) but
> **vanished** after changes to `guardRandomization` (`0.5`, out of range), `size` (`1.5`), and
> budget. Notably, **raising the budget back to `300000` did not restore them**, so the guarded
> budget was *not* the cause — likely the larger `size`, the out-of-range randomization, or RMG
> variance. The probe is now reverted to the known-good config (`size 1.0`, budget `200000`) with
> **only `guardRandomization` differing** (A `0.0`, B `0.25`). RMG placement still varies — if a zone
> lacks the six, **regenerate once or twice** before concluding.

**✓ Confirmed:** with the reverted config both zones generated the six objects; Neutral-A (`0.0`) had
uniform guard sizes while Neutral-B (`0.25`) had clearly varied sizes — `guardRandomization` is a
per-guard ± spread on guard values.

### Test 13 — zero / per-area content budgets — [`Doc-Probe-ZeroBudget.rmg.json`](test-templates/Doc-Probe-ZeroBudget.rmg.json)
Resolves the Q5 budget questions. A 2-player chain with **three** neutral zones, all sharing the same
guarded pool and having **no guarded mandatory content** (only unguarded mines), so any guarded
object present must have come from the pool. They differ only in the guarded budget:
- **Neutral-Zero** — `guardedContentValue: 0` **and** `guardedContentValuePerArea: 0` (the *Symmetry* case)
- **Neutral-PerArea** — `guardedContentValue: 0`, `guardedContentValuePerArea: 2000` (per-area only)
- **Neutral-Ctrl** — `guardedContentValue: 150000` (control)

Walking from **Player 1**, the order is **Neutral-Zero → Neutral-PerArea → Neutral-Ctrl** (Player 2).
All three should still show their unguarded mines + some unguarded/resource content, so a zone that
generates but has **no guarded objects** is meaningful, not broken.

**Observe & report — count the *guarded* objects (banks/buildings with guard stacks) in each:**
- **Neutral-Zero:** any guarded objects at all? If **none**, a both-zero guarded budget means the
  pool contributes nothing (only mandatory/explicit content appears — explaining how *Symmetry*
  works with all-zero values). If some appear anyway, zero is a "use default" sentinel.
- **Neutral-PerArea:** does guarded content appear here (proving `guardedContentValuePerArea` alone
  drives content)? Roughly how much vs. the control?
- **Neutral-Ctrl:** should have guarded content (sanity check).

(If a zone is sparse, regenerate once or twice — RMG placement varies.)

**✓ Confirmed:** Neutral-Zero (both-zero) → **no guarded content** (only mandatory unguarded mines +
a little unguarded/resource content). Neutral-PerArea (`perArea: 2000` only) → guarded content
present (so per-area alone works, area-scaled — it had *more* than the control). Neutral-Ctrl
(`value: 150000`) → guarded content present. Only the both-nonzero combination rule is left untested.

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
