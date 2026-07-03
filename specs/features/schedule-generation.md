# Schedule Generation

**Path:** features/schedule-generation
**Scope:** How `generateSchedule(seed)` produces a fully-resolved `ScheduleEntry[]` from a seed — cadence curve, phase transitions, category selection, placement, and solvability veto integration. Excludes entity behavior at runtime (see architecture.md) and tech stack rationale (already decided).

## Density curve — phase-ramped intervals, no jitter

Spawn timing is governed by per-phase target intervals with cosine-ease ramp transitions between them. No per-spawn timing jitter is applied — same-seed replays produce identical spawn timing, full stop.

### Phase targets

| Phase | Window | Target interval | Intended feel |
|---|---|---|---|
| Naive Room | 0:00–0:45 | 8.0 s | "this is cute" |
| The Fill | 0:45–2:00 | 5.0 s | leaning in |
| Crowding | 2:00–3:15 | 3.0 s | sweating |
| The Flood | 3:15–4:30 | 1.5 s | panic |
| Last Words | 4:30–4:55 | 1.0 s | maximum chaos |
| Last Words silence | 4:55–5:00 | ∞ (no spawns) | held breath |
| The Crossing | 5:00+ | ∞ (spawning stops) | the run that matters |

### Ramp transitions

At each phase boundary, the interval ramps from the previous phase's target to the new phase's target over a transition window using cosine easing (smooth start and end, no harsh inflection). This eliminates the jarring instant rate-doubling that a hard step would produce at phase boundaries (e.g. 3.0s → 1.5s at 3:15) while preserving the GDD's authored "the phase just shifted" feel.

```typescript
const RAMP_SECONDS = 8; // tune empirically — transition window at each phase boundary

function getInterval(t: number): number {
  const phase = getPhaseAt(t);
  const prevInterval = phase.index === 0 ? phase.interval : phases[phase.index - 1].interval;
  const elapsed = t - phase.startTime;
  const rampDuration = Math.min(RAMP_SECONDS, phase.duration);
  const rampProgress = Math.min(1, elapsed / rampDuration);
  // Cosine ease: smooth start and end
  const eased = 0.5 * (1 - Math.cos(Math.PI * rampProgress));
  return lerp(prevInterval, phase.interval, eased);
}
```

### Why no jitter (rejected alternative)

Per-spawn timing jitter (e.g. ±20% randomized offset on each interval) was considered and rejected. The GDD's "~8s" notation is design-time imprecision (the designer's rough target), not an instruction to inject runtime randomness. Jitter would make same-seed replays produce non-identical spawn timing, which directly undermines:

- **Daily Bin leaderboard fairness** — players competing on the same seed must face the identical spawn sequence so improvement is measurably about execution, not about getting a luckier jitter roll on attempt three
- **Classic mode learnability** — the GDD says "the room is learnable, mastery is memory + execution" (§2.6); exact timing is part of what's learned
- **Prior art** — Vampire Survivors uses exact, learnable wave timing as a documented feature; daily-seed roguelites (Spelunky Daily, Nuclear Throne dailies) rely on identical generation for competitive integrity

### Estimated total spawns

Under the ramped-interval model: ~115–125 total spawn events across 5 minutes. Some entities are transient (despawn on timer, collected, swept by Intern's Broom), so the saturated object count at 5:00 is lower — consistent with the GDD's ~107 figure (§6.1).

## ScheduleEntry shape

Schedule entries split into two kinds before any type-level discrimination — entity spawns (spatial, have a position) and global events (non-spatial, affect game state). This mirrors the `EntityData` two-level discriminant pattern from architecture.md (Answer 5) and prevents forcing meaningless `position` fields onto non-spatial entries like Coffee Break.

### Type structure

```typescript
interface ScheduleEntryBase {
  scheduledTick: number;       // exact tick from ramped interval — distinct from EntityBase.spawnTick
  telegraphTicks: number;      // 42 ticks (0.7s) default, tunable per type
}

// --- Entity spawns (spatial) ---

type EntitySpawnEntry = ScheduleEntryBase & { kind: 'entitySpawn' } & EntitySpawnParams;

type EntitySpawnParams =
  // Fully pre-rolled — all parameters known at generation time
  | { entityType: 'mysteryCrate'; position: Point; contents: EntityType }
  | { entityType: 'gachaDoor'; position: Point; teleportTarget: Point }
  | { entityType: 'weatherVane'; position: Point; modifier: WeatherModifier }
  | { entityType: 'dice'; position: Point; face: DiceFace }
  | { entityType: 'timeTokenMinus'; position: Point; delta: -10 }
  | { entityType: 'timeTokenPlus'; position: Point; delta: 10 }
  | { entityType: 'buoyPlatform'; position: Point; floatHeight: number }
  // Reject variants are distinct EntityTypes, not a boolean flag on the base power-up.
  // e.g. 'doubleJump' and 'doubleJumpReject' are separate entries in the type-stream
  // weighted table — one draw selects between them, no follow-up boolean check.
  | { entityType: 'doubleJump'; position: Point }
  | { entityType: 'doubleJumpReject'; position: Point }
  // ... remaining fully pre-rolled types (each with position + type-specific params)

  // Pre-rolled trigger, runtime-resolved content — spawn timing and position are
  // pre-rolled, but the actual entity that manifests depends on GameState at spawn time.
  // Resolution happens in processScheduledSpawns() during step(), not at generation time.
  | { entityType: 'regifter'; position: Point; tieBreakDraw: number }
  | { entityType: 'understudy'; position: Point }
  | { entityType: 'complaintForm'; position: Point }
  // ... other types whose content reads runtime logs (death log, avoidance log)
  ;

// --- Global events (non-spatial) ---

type GlobalEventEntry = ScheduleEntryBase & { kind: 'globalEvent' } & GlobalEventParams;

type GlobalEventParams =
  | { eventType: 'coffeeBreak'; durationTicks: number }
  | { eventType: 'lastWordsSilence' }    // marks the 5s silence before 5:00
  ;

type ScheduleEntry = EntitySpawnEntry | GlobalEventEntry;
```

### Coffee Break schedule

Coffee Breaks are **fixed-time global events**, not probabilistic draws. Two breaks per session at deterministic times:

| Break | Scheduled time | Duration |
|---|---|---|
| First | ~1:30 (5,400 ticks) | 10s (600 ticks) |
| Second | ~3:00 (10,800 ticks) | 10s (600 ticks) |

Times are exact (not approximate) once locked — the GDD's "~" notation is design-intent imprecision, same as the spawn-interval targets. `generateSchedule()` places these as `GlobalEventEntry` items with `kind: 'globalEvent'` at the specified ticks. They do not consume any PRNG draws.

During a Coffee Break: spawning pauses (no new `ScheduleEntry` items materialize), all moving hazards freeze, static hazards remain active. The `coffeeBreak` field in `GameState` tracks active state and remaining ticks.

### Three resolution categories

Not all schedule entries are fully resolved at generation time. The categories, with their resolution points:

| Category | Resolved at | Examples | What's pre-rolled | What's runtime |
|---|---|---|---|---|
| **Fully pre-rolled** | `generateSchedule()` | Mystery Crate, Gacha Door, Weather Vane, Dice, all standard entity types | Everything — entityType, position, all params | Nothing |
| **Pre-rolled trigger, runtime-resolved content** | `processScheduledSpawns()` in `step()` | Regifter, Understudy, Complaint Form | Spawn timing, position, tie-break fallback | Which entity actually manifests (reads avoidance/death log) |

**Runtime-resolved entity behavior specs (deferred):** Regifter, Understudy, Complaint Form, and Intern's Broom have their *architecture integration points* specified here (spawn category, log-read-timing rule, tie-break draw) and in architecture.md (step ordering, deferred-removal). Their *gameplay behavior* (what each entity does once spawned) is defined in the GDD (§4–5) and will be specified in per-entity spec files when those entities enter implementation scope. They are not in the prototype plan.
| **Fully runtime** | Inside `step()`, not on schedule | Monument | Nothing pre-rolled — triggered by anti-idle rule | Everything — position is player location, existence depends on solvability veto |

This is a named pattern, not an ad-hoc exception list. When adding a new entity type, determine which category it belongs to by asking: "does this type's *identity* or *parameters* depend on runtime player behavior?" If yes, it's category 2 or 3. If no, it's category 1.

### Naming: `scheduledTick` vs `spawnTick`

`ScheduleEntry.scheduledTick` (generation-time intended tick) and `EntityBase.spawnTick` (actual tick the entity was created, from Answer 6) are conceptually different fields that usually have the same value. They can diverge for runtime-resolved entries (Regifter's content is determined at `scheduledTick` but the entity type written to `spawnTick` may differ from what was scheduled). Named differently to prevent implicit conflation.

### Gacha Door teleportTarget reachability

`teleportTarget` is pre-rolled at generation time from the chaos stream. A reachability check validates that the destination is reachable-to-door: `checkSolvability(permanentEntities, teleportTarget, SHIP_IT_DOOR)` must pass. On failure, re-roll the `teleportTarget` from the chaos stream until it passes. This prevents a player being teleported into a stranded pocket with a functionally bricked run.

## PRNG stream assignment

Four sub-streams (timing stream removed — see below). Draw order within each spawn event is fixed and load-bearing for determinism.

### Stream map

| Decision | Stream | Draws per spawn | Notes |
|---|---|---|---|
| **What** — entity category + specific type | `type` | 2 (category, then type-within-category) | Two-stage weighted draw: first draw selects category from phase budget, second draw selects specific EntityType within that category. Both draws skipped (zero consumption) when a guarantee override triggers. |
| **Where** — position | `placement` | 1+ (variable) | Generates (x, y) candidate. May re-draw on constraint rejection (spawn pocket, reserved corridor, minimum spacing). Variable consumption rate is a known property. |
| **Chaos parameters** | `chaos` | 0+ (type-dependent) | Mystery Crate contents, Gacha Door target, Weather Vane modifier, Dice face, Regifter tie-break. Zero draws for simple types. |
| **Visual jitter** | `cosmetic` | 0 at generation | Runtime-only. Never consumed during schedule generation. |

### Why no timing stream

Spawn timing is fully determined by the ramped interval curve — `nextTick = previousTick + getInterval(previousTime) * 60`. No PRNG draw is involved. The tech-stack's original five sub-streams are revised to four.

**Per-category minimum-frequency guarantees** (e.g., Buoy Platform ≥1/45s in The Flood) are enforced as deterministic override rules — force `entityType` at the last eligible slot before the window closes, zero RNG, zero timing-stream consumption. This was audited during the phase-budget grilling session: all known guarantees (windowed-recurring and first-occurrence) are deterministic overrides that bypass the type-stream draws entirely. The timing stream remains removed.

### Draw order per spawn event (named rule)

```
For each spawn event:
  1. Compute candidate scheduledTick from ramped interval curve (zero RNG)
  2. Check guarantee overrides — if triggered, force entityType, skip steps 3–4 (zero type-stream draws)
  3. type.next()      → select category from phase-weighted budget table
  4. type.next()      → select specific entityType within category
  5. placement.next() → generate position candidate, apply placement pipeline (see below)
  6. chaos.next()     → fill type-specific params (zero draws for simple types)
  7. Telegraph-cap check → if committing at candidate tick would exceed 2 concurrent telegraph windows,
     delay scheduledTick forward until it clears (deterministic, zero RNG)
```

This order is load-bearing. Steps 3–4 must be sequential draws from the same `type` stream (not separate streams). If a guarantee override triggers at step 2, **both** type-stream draws are skipped for that slot — the stream cursor does not advance, ensuring downstream spawns are unaffected.

### Seed-invariance: two distinct claims

These were conflated in an earlier draft and must be stated separately:

- **The base ramped-interval cadence curve is seed-invariant** — `getInterval(t)` is a pure function of elapsed time with zero RNG input. True unconditionally.
- **The final committed `scheduledTick` values are NOT seed-invariant** — telegraph-cap delays (step 7) and guarantee overrides (step 2) both introduce seed-dependent tick drift on top of the universal base curve. Two seeds may produce slightly different committed tick sequences even though their base cadence curves are identical.

### Phase-boundary rule for delayed spawns

If step 7's telegraph-cap delay pushes a spawn's `scheduledTick` past its original phase boundary into the next phase, the spawn **keeps its original type/category draw** (steps 3–4 already consumed from the type stream under the old phase's weights). Re-drawing under the new phase's table would consume extra, seed-dependent draws that complicate the stream-consumption model. A few-tick boundary drift is not worth re-rolling for.

### Telegraph-cap enforcement

The GDD's "≤2 spawns telegraph at once" constraint is **global and temporal**, not spatial. The entire 960×540 arena is visible simultaneously (no camera scroll, per tech-stack's `Phaser.Scale.FIT` at 960×540), so every telegraph is visible to the player regardless of position.

At Last Words cadence (~1s intervals) with 42-tick (0.7s) telegraph duration, consecutive telegraph windows naturally overlap. A third spawn landing inside the same window is plausible and would break the cap. This constraint is actively load-bearing, not naturally satisfied by the cadence design.

**Resolution: delay `scheduledTick` forward.** When committing a spawn would put >2 concurrent telegraph windows active, push its `scheduledTick` forward tick-by-tick until committing would not exceed the cap. This is the only resolution consistent with not silently sacrificing hazard-warning fairness (suppressing the telegraph) or violating single-screen spatial logic (shifting position).

No strong external genre precedent was found for simultaneous-telegraph caps specifically. This resolution is derived from the project's own constraints and should be validated in playtesting as a first-principles design decision.

## Two-stage type selection

Type selection uses a two-stage weighted draw, following the standard loot-table pattern (Diablo-style ARPG precedent): one draw resolves the category/tier, a second draw resolves the specific member within it. This keeps category-level budget tuning independent of per-type weights.

### Phase category budget tables

```typescript
interface PhaseWeights {
  categoryWeights: Record<EntityCategory, number>;  // sum to 1.0
  typeWeights: Record<EntityCategory, Record<EntityType, number>>;  // per-category, sum to 1.0 within each
}

const PHASE_WEIGHTS: Record<Phase, PhaseWeights> = { /* see table below */ };
```

| Category | Naive Room | The Fill | Crowding | The Flood | Last Words |
|---|---|---|---|---|---|
| `hazard` | 50% | 55% | 40% | 35% | 30% |
| `platform` | 25% | 15% | 12% | 10% | 5% |
| `collectible` | 15% | 10% | 8% | 8% | 5% |
| `reliefItem` | 10% | 8% | 10% | 5% | 0% |
| `powerUp` | 0% | 7% | 10% | 10% | 10% |
| `enemy` | 0% | 5% | 10% | 15% | 20% |
| `chaoticSpawn` | 0% | 0% | 10% | 17% | 30% |

**Column notes:**
- **Naive Room** — matches GDD verbatim (50% hazard, 40% platform/collectible, 10% relief). Zero enemies/chaos/power-ups — GDD says "first enemy" arrives in The Fill.
- **The Fill** — GDD's "hazards ramp to 55%." Introduces enemies at 5%, power-ups at 7%. First Mystery Crate arrives via first-occurrence guarantee override, not category budget.
- **Crowding** — GDD's "enemies + chaos to 25%" split 10/15. Relief gets a pulse (10%, up from 8%) because GDD explicitly says "relief pulses before density spikes."
- **The Flood** — GDD's "everything." All categories active. Relief drops to 5% ("Bus Stops rarer").
- **Last Words** — "maximum chaos." `chaoticSpawn` peaks at 30%. Relief drops to 0% — no mercy in the final 25 seconds.

`misc` is excluded from the budget table — player entity and monuments are not schedule-spawned.

**Important: `platform` in this table governs special platform variants** (Buoy Platform, Ghost Scaffold, Apologetic Platform, etc.) layered on top of the base-traversal guarantee (see below). It does not govern floor geometry the player needs for basic reachability. Without this distinction, `platform` at 5% during Last Words could produce unreachable seeds — a broken, unwinnable pre-rolled schedule that still passes replay verification.

### Base-traversal guarantee

Minimum reachable platform spacing is generated as a **guaranteed layer before the budget table runs**, independent of category weights. This ensures every seed has a navigable path from spawn pocket to SHIP IT door regardless of how the budget table distributes its weights.

The guarantee reuses the same `checkSolvability()` flood-fill already specified for the solvability veto — applied to the base traversal layer at generation time, before any budget-drawn platforms are placed on top. The `platform` category budget then governs special platform types layered over this guaranteed base.

This is a structural separation, not a tuning knob: even at `platform` = 0%, a seed must be traversable.

#### Generation pipeline ordering (named rule)

The two layers write to the **same** committed-entities array in strict sequence, not as independent passes:

1. **Base-traversal pass:** Generate minimum-reachable platform positions. Each committed position is appended to the `committedEntities` array and marked as a reserved tile in the nav grid. `checkSolvability()` validates the complete base layer.
2. **Budget-table pass:** For each spawn slot, the two-stage type draw and placement pipeline run against the `committedEntities` array that already contains all base-traversal positions. Placement collision checks (AABB overlap, minimum spacing) see base-traversal entities as occupied space — no double-booking possible.

A budget-drawn platform landing at the same position as a base-traversal platform would fail the AABB-overlap check in step 2a of the placement pipeline and re-draw. There is no separate "reservation" data structure — the single `committedEntities` array is the reservation.

### Category composition: step at phase boundaries

Category mix **steps** (instant change) at phase boundaries, not ramps. The cadence (spawn interval) already ramps smoothly via cosine-ease — having category mix *also* ramp would blur the "the room just shifted" feel the GDD wants each phase to produce. A sudden shift in *what* spawns combined with a smooth shift in *how fast* gives the clearest phase-identity signal.

### Within-category type weights

**Flat-equal default** across all types within each category, with tuning deferred to playtesting. This isolates category-level pacing as a separate, testable question from type-level variety. Reject variant weights are also flat-equal with their base type (e.g., `doubleJump` and `doubleJumpReject` each get 50% of their combined weight slot) — tunable after playtesting reveals whether reject frequency feels right.

### Phase-gating rules

Types with zero weight before their first eligible phase:

| Type | First eligible phase | Reasoning |
|---|---|---|
| All enemies | The Fill | GDD: "first enemy" in The Fill |
| Mystery Crate | The Fill | GDD: "first Mystery Crate" in The Fill |
| Gacha Door | Crowding | High-risk teleport too chaotic for early game |
| Weather Vane / Dice | Crowding | Room-wide modifiers need enough objects to matter |
| Regifter / Understudy | Crowding | Need death/avoidance history to have meaningful content |
| Buoy Platform | The Fill | Needs floor hazards to float above |
| Ghost Scaffold | The Fill | Needs airborne gameplay to justify existence |
| Reject variants | The Fill | New players need to learn the real version first |
| Time Token (−10s) | Crowding | GDD: "rare, deep in danger" — too early in Naive/Fill |
| Time Token (+10s) | The Fill | The room's "favorite lie" — introduce the deception early |

Every type in this table has an associated **first-occurrence guarantee** (see below) ensuring it actually appears within a bounded window of its first eligible phase, not just that it *can* appear.

## Guarantee overrides

Certain types have hard minimum-frequency constraints that override the probabilistic two-stage draw. Two distinct override shapes exist:

### Windowed-recurring (tracks recency)

A type must appear at least once per rolling time window within specified phases. Checked at each spawn slot; if the window has elapsed without an occurrence, the type is forced for this slot.

```typescript
interface WindowedGuarantee {
  entityType: EntityType;
  windowTicks: number;           // e.g. 45s * 60 = 2700 ticks
  activeInPhases: Phase[];
  lastSpawnTick: number;         // updated on every occurrence (by any means — draw or override)
}
```

**Known instance:** Buoy Platform ≥ 1 per 45s in The Flood.

### First-occurrence (tracks whether it’s happened at all)

A type (or category) must appear at least once within a bounded window after becoming eligible. Checked at each spawn slot; if the eligibility window is about to close without an occurrence, the type is forced.

```typescript
interface FirstOccurrenceGuarantee {
  entityType: EntityType | EntityCategory;  // can guarantee a specific type or any member of a category
  mustOccurByPhase: Phase;
  graceTicks: number;            // max ticks into the eligible phase before forcing — default: phase duration
  hasOccurred: boolean;          // set to true on first occurrence (by any means — draw or override)
}
```

**Generalized rule:** Every phase-gated type in the phase-gating table above gets a first-occurrence guarantee. Without this, "first eligible phase: The Fill" is a ceiling on when a type *can* appear, not a guarantee it *will* appear early — bad luck could push a player's first enemy sighting to the very end of The Fill, weakening the intended pacing beat.

**Known instances (all rows from the phase-gating table):**

| Guarantee | Must occur by | Grace window |
|---|---|---|
| First enemy (any) | The Fill | Within 30s of The Fill start |
| First Mystery Crate | The Fill | Within 45s of The Fill start |
| First Buoy Platform | The Fill | Within 60s of The Fill start |
| First Ghost Scaffold | The Fill | Within 60s of The Fill start |
| First reject variant (any) | The Fill | Within 75s of The Fill start |
| First Time Token (+10s) | The Fill | Within 45s of The Fill start |
| First Gacha Door | Crowding | Within 30s of Crowding start |
| First Weather Vane or Dice | Crowding | Within 45s of Crowding start |
| First Regifter or Understudy | Crowding | Within 60s of Crowding start |
| First Time Token (−10s) | Crowding | Within 45s of Crowding start |

Grace window values are tuning parameters — expect adjustment after playtesting. The mechanism is the same reusable `FirstOccurrenceGuarantee` structure, not per-type special-case code.

#### Per-type independent tracking (named rule)

Each `FirstOccurrenceGuarantee` instance tracks its own `hasOccurred` flag independently. Guaranteeing an early enemy does **not** satisfy or interact with the Mystery Crate guarantee — they are separate counters checked separately at each spawn slot. If multiple guarantees are eligible to fire at the same slot, **priority order** is: earliest `graceTicks` deadline first (most urgent fires first). The losing guarantee's window continues ticking; it fires at its own deadline or when a natural draw satisfies it, whichever comes first.

### Override RNG impact

Both override types are deterministic (zero-RNG). When triggered, they bypass the two-stage type draw entirely — **both** the category draw and the specific-type draw are skipped, so the `type` stream advances by zero draws for that slot. This ensures downstream spawn sequences are unaffected by whether an override fired.

### "Bus Stops rarer" is not a guarantee

The GDD's "Bus Stops rarer" in The Flood is a weight reduction (lower within-category probability), not a minimum-frequency guarantee. No override mechanism needed — handled by the phase weight table.

### Solvability-veto relocation — separate from placement re-rolls

When the solvability veto rejects a permanent spawn's position, relocation is a **deterministic nearest-legal-position search** (expanding-radius scan or BFS on the nav grid from the rejected coordinate) — not a random re-draw from the `placement` stream. The GDD's language ("relocates to its nearest legal position") implies a metric-guided search with a well-defined answer, not a chance-based re-roll.

This distinction matters: random re-rolls don't find the *nearest* legal position, they find *some* legal position after a variable number of draws, consuming the placement stream at a rate that depends on the current entity layout. Solvability relocation consumes zero RNG draws, same as the timing computation — it's a deterministic correction, not a random search.

## Placement pipeline

The `placement` stream generates a candidate `(x, y)`. A pipeline of filters then accepts or rejects it. Cheapest/most-foundational checks first:

```
1. Bounds: spawn pocket exclusion (x < 110), reserved corridor exclusion for permanent types (x > 895) — O(1)
2. Minimum spacing:
   a. AABB overlap (all types) — candidate must not physically overlap any entity committed at or before this scheduledTick
   b. Jump-arc clearance buffer (platform + hazard categories only) — minimum gap between AABBs sized as a
      fraction of the existing jump-arc constants (105px apex / 140px flat reach), e.g. ~35–45px.
      Reuses the project's established navigation metric rather than inventing an arbitrary constant.
      Tunable design knob, but derived from jump-arc, not ad hoc.
   Both checks are O(n) against currently-committed entities.
3. Solvability veto (permanent spawns only): full flood-fill reachability check. On failure, deterministic
   nearest-legal-position search (zero RNG). Most expensive, runs last among spatial checks.
```

If all spatial checks fail after a bounded number of re-draws from the `placement` stream (max `PLACEMENT_MAX_REDRAWS = 10`, tunable), the spawn is placed at the solvability veto's nearest-legal fallback position (which always exists as long as the path isn't fully bricked, guaranteed by sequential veto ordering). The re-draw limit is a named constant, not a magic number — it caps PRNG consumption to a predictable rate per spawn slot.

**Fallback behavior when cap is hit:** The fallback is always the solvability veto's nearest-legal-position search (zero RNG, deterministic). This position is guaranteed to exist by sequential veto ordering — if it didn't, an earlier spawn in the sequence would have been vetoed first. The cap is not an error condition; it's a performance/determinism bound on the placement stream. No reseed, no thrown error, no relaxed constraints — just the deterministic fallback that was always the terminal case.

## Acceptance criteria

- [ ] `generateSchedule(seed)` is a pure function: same seed → identical `ScheduleEntry[]`, no external state reads
- [ ] Spawn timing is exactly reproducible across runs with the same seed — no per-spawn randomization on intervals
- [ ] Phase transitions use cosine-ease ramp, not hard steps — verified by plotting spawn density over time for a test seed
- [ ] Total spawn count across 5 minutes falls within 110–130 range (tune via phase intervals and ramp duration)
- [ ] Last Words silence (final 5s before 5:00) produces zero spawn events
- [ ] The Crossing (5:00+) produces zero spawn events
- [ ] `ScheduleEntry` uses `kind: 'entitySpawn' | 'globalEvent'` discriminant — no `position` field on global events (Coffee Break, Last Words silence)
- [ ] Reject variants are distinct `EntityType` members in the type-stream weighted table — no `isRejectVariant` boolean flag
- [ ] Runtime-resolved entries (Regifter, Understudy, Complaint Form) carry only pre-rollable fields at generation time — actual entity type is resolved in `processScheduledSpawns()` from runtime state
- [ ] Gacha Door `teleportTarget` passes `checkSolvability()` at generation time — destination must be reachable-to-door; re-roll on failure
- [ ] Draw order per spawn event is: scheduledTick (zero RNG) → guarantee check → type (2 draws: category then specific) → placement → chaos — no reordering without full schedule-impact review
- [ ] Guarantee overrides skip both type-stream draws (zero consumption) for the overridden slot
- [ ] Windowed-recurring guarantees (Buoy Platform) track recency and force type at window expiry
- [ ] First-occurrence guarantees exist for every phase-gated type — each forces its type within the specified grace window of its first eligible phase
- [ ] Category budget tables match the specified percentages (all columns sum to 100%) and step (not ramp) at phase boundaries
- [ ] Base-traversal guarantee produces a navigable path from spawn pocket to SHIP IT door independent of category budget — verified by generating 100 seeds and running `checkSolvability()` against the base layer alone
- [ ] Phase-gated types have zero weight before their first eligible phase
- [ ] Within-category type weights default to flat-equal — no hand-tuned per-type weights without playtesting evidence
- [ ] Solvability-veto relocation uses deterministic nearest-legal-position search (zero RNG draws) — never consumes the `placement` stream
- [ ] Placement-constraint rejections (overlap, spacing) re-draw from `placement` stream — distinct mechanism from solvability relocation
- [ ] Minimum spacing includes both AABB non-overlap (all types) and jump-arc-derived clearance buffer (platform/hazard categories) — buffer sized as fraction of 105px/140px jump-arc constants
- [ ] Telegraph-cap enforcement delays `scheduledTick` forward when committing would exceed 2 concurrent telegraph windows — deterministic, zero RNG, globally scoped (single-screen arena)
- [ ] Telegraph-cap-delayed spawns keep their original phase's type/category draw even if pushed past a phase boundary
- [ ] Per-category minimum-frequency guarantees (Buoy Platform ≥1/45s in The Flood) are enforced via deterministic override rule, not RNG — audited against full GDD catalog before finalizing
- [ ] Base-traversal entities are committed to the same `committedEntities` array before budget-table rolls — budget placement sees them as occupied space via AABB overlap check (no double-booking)
- [ ] Each `FirstOccurrenceGuarantee` tracks `hasOccurred` independently per type — satisfying one guarantee never satisfies another
- [ ] Golden-replay fixtures include at least one fixture per phase boundary (Naive→Fill, Fill→Crowding, Crowding→Flood, Flood→LastWords) snapshotting state at the exact transition tick — catches step-discontinuity off-by-one bugs in category composition switching

## Dependencies

- [specs/features/architecture.md](architecture.md) — Generation Output, GameState, Session factory
- [docs/adr/0005-pre-roll-all-outcomes.md](../../docs/adr/0005-pre-roll-all-outcomes.md)
- [specs/features/tech-stack.md](tech-stack.md) — PRNG sub-streams, SFC32
