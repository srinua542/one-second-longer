# Architecture / System Design

**Path:** features/architecture
**Scope:** Component interaction — game loop, entity model, state ownership, physics, rendering. Excludes tech stack rationale (already decided) and gameplay balancing.

## Game loop

| Layer | Responsibility |
|---|---|
| `simulation/` | Owns `step(state, generatedRun, input, FIXED_DT, events?)` — deterministic, mutates `state` in place, pushes to caller-owned `events` array if provided, no wall-clock, no rendering |
| `client` | `GameScene.update(time, delta)` accumulates `delta`, allocates one `SimEvent[]` per frame, calls `simulation.step()` in a while-loop (accumulator pattern) capped at `MAX_TICKS_PER_FRAME`, then consumes events for view sync + cosmetic triggers |
| `server` | Headless loop calls `simulation.step(FIXED_DT)` per recorded input frame, passes no events collector — zero allocation overhead |

- **FIXED_DT** = 1/60s, constant across client and server
- **MAX_TICKS_PER_FRAME** = 5 (tune empirically) — hard cap on simulation ticks per render frame to prevent the spiral-of-death (Fiedler's "Fix Your Timestep!"). If the accumulator still has leftover time after the cap, **drop it** (reset accumulator to 0) rather than carrying the backlog forward. Rationale: "catch up slowly" would fast-forward the game after a stall, which is jarring for a skill-based platformer and unfair for Daily Bin leaderboard runs (a tab-backgrounding event would silently advance the player through hazard spawns they never reacted to). Dropping time is consistent with the portal-pause pattern — a stall is treated as an unintentional pause, not lost game time.
- Rendering (client only) runs at variable framerate; interpolation between last two simulation states is optional polish, not required for correctness
- Simulation never reads `Date.now()`, `performance.now()`, or any wall-clock — only tick count

### Client accumulator pseudocode

```typescript
const FIXED_DT = 1 / 60;
const MAX_TICKS_PER_FRAME = 5;

function update(time: number, delta: number) {
  accumulator += delta / 1000; // Phaser delta is in ms
  const events: SimEvent[] = [];
  let ticksThisFrame = 0;

  while (accumulator >= FIXED_DT && ticksThisFrame < MAX_TICKS_PER_FRAME) {
    simulation.step(session.state, session.run, currentInput, FIXED_DT, events);
    accumulator -= FIXED_DT;
    ticksThisFrame++;
  }

  // Drop remaining backlog — don't carry forward a debt that causes fast-forward
  if (accumulator >= FIXED_DT) {
    accumulator = 0;
  }

  // Process ALL events in array order — never reorder, never deduplicate
  for (const event of events) {
    handleSimEvent(event);
  }
}
```

### Event processing contract

- The client **must** process events in array order, never reorder or deduplicate — events from multiple ticks within a single frame are interleaved, and sequential replay is the only correct consumption strategy
- A `GameObjects` sprite created and destroyed within the same `update()` call (before Phaser's render pass) is never drawn — the redundant allocation/deallocation cost is near-zero
- Tick-boundary markers in the event stream (Option B from the design discussion) are **deferred, not rejected** — revisit only if a concrete consumer needs tick-level batching (e.g. SFX deduplication during multi-tick frames). The trigger for revisiting is documented, not left implicit

## Input

### Input type

```typescript
interface Input {
  left: boolean;
  right: boolean;
  jump: boolean;  // held state (level signal), not press edge — simulation detects edge internally
}
```

`jump` is a **held-state boolean** — `true` means the button is currently down. The simulation tracks the previous tick's held state in `PlayerState.prevJumpHeld` and detects the press edge (`!prev && current`) internally. This keeps the input abstraction dead simple (three booleans representing physical button state) and puts all timing logic (coyote time, jump buffer) in `simulation/` where it's deterministic and replay-verifiable.

### Sampling

Input is sampled **once per frame**, before the accumulator loop. All ticks within the same frame receive the same `Input` snapshot. This matches Phaser's own input polling model (`preUpdate` before `update`) and avoids the impossible task of sampling input between simulation ticks that execute synchronously within a single JS event loop turn.

**Known limitation (accepted):** An extremely short press-and-release that both happen within a single stuttered multi-tick frame can be missed, since only one snapshot is taken per frame. This is inherent to polling-based input and consistent with the drop-time philosophy from the accumulator cap — not a bug, not worth solving (event-queue capture would reintroduce per-tick input assignment ambiguity).

### Input log format

One `Input` entry per simulation tick — a flat array. During a multi-tick frame, the same `Input` is pushed N times. Length of `inputLog` equals total ticks in the run. Server iterates `inputLog[i]` for tick `i`.

A 5-minute run at 60 FPS = 18,000 entries × 3 booleans = trivially small. No compression needed for submission.

```typescript
// Client-side, inside update():
const input = inputAbstraction.sample(); // once per frame, before loop
while (accumulator >= FIXED_DT && ticksThisFrame < MAX_TICKS_PER_FRAME) {
  inputLog.push(input);  // same reference — Input is treated as immutable
  simulation.step(session.state, session.run, input, FIXED_DT, events);
  accumulator -= FIXED_DT;
  ticksThisFrame++;
}
```

## Entity model

- `simulation/` defines entities as **plain serializable data** — shared spatial fields on `EntityBase`, type-specific fields on a discriminated `EntityData` union (Option C: shared base + typed discriminant)
- Two-level discriminant: `category` (behavioral group — platform, hazard, enemy, etc.) and `type` (specific kind — buoyPlatform, grudgePlank, magpie, etc.). System functions narrow by `category` first to work with a small sub-union, then by `type` only where per-type variation exists
- No inheritance hierarchy; behavior lives in system functions (`updateHazards(state)`, `updateEnemies(state)`) operating over entity arrays — ECS-lite, not full ECS
- Every exhaustive `switch` over `type` or `category` ends with an `assertNever(e)` guard — new entity types added to the union produce compile errors at every call site that needs updating
- Client maintains a parallel `EntityView` map: `entityId → Phaser.GameObjects.*` — needs only `EntityBase` fields (`x, y, type`) for positioning

### View lifecycle — events primary, diff as bootstrap

**Primary mechanism (per-frame):** `SimEvent` emissions from `step()` drive view creation/destruction and cosmetic triggers. Events carry causality — a `'death'` event names the hazard that killed, an `'entityDespawned'` event tells you whether it timed out or was swept by the Intern's Broom — so the client can pick the right VFX/SFX without reverse-engineering diffs.

**Secondary mechanism (bootstrap + backstop):** On scene creation/recreation (returning from portal pause, ad break), the client has no event history. It reconciles `EntityView` keys against current `entities[]` IDs as a one-time diff to bootstrap all views. This same diff may run as a cheap periodic consistency check (O(n) over a small array) to catch bugs where a mutation site forgets to emit an event. This is a safety net, not the per-frame driver.

- Each view is a dumb puppet: `view.setPosition(entity.x, entity.y)` — no logic, no physics body driving position
- Cosmetic-only Arcade Physics objects (debris, particles) are **not** simulation entities — they live entirely in `client`, spawned by reacting to `SimEvent` emissions (e.g. `'hazardTriggered'` → spawn particle burst), never reported back

## State ownership — three-layer model (ADR-0008)

All simulation data is split into three layers with distinct lifecycles:

| Layer | What it is | Lifecycle | In replay hash? |
|---|---|---|---|
| **Generation Output** | Pre-rolled schedule (spawn timing/type/placement/chaos, per ADR-0005) | Computed once from `seed` at run start, on both client and server, from the same RNG code path | No — regenerated from `seed`, not stored |
| **GameState** (canonical) | Everything with memory that affects future ticks or the final score | Mutated every `simulation.step()` | Yes — this *is* what gets hashed |
| **Derived / View** | Anything computable from GameState + Generation Output with no memory of its own | Recomputed on demand, every read | No — recomputing must be cheap and pure |

### GameState contents

| Field | Reasoning |
|---|---|
| Entity positions/velocities | Mutates every tick, feeds collision next tick |
| Schedule cursor (next-spawn-index) | Mutable pointer into the immutable schedule |
| Phase progression | Has memory (current phase, when it started), gates behavior |
| Score / deaths | Directly the verified output |
| Anti-idle rolling window | Ring buffer with memory that gates idle penalty |
| Death log | Read by Complaint Form and Understudy — gameplay-authoritative |
| Avoidance log | Read by Regifter — gameplay-authoritative |
| Coffee Break status | Mode flag + remaining duration |
| Active power-up timers | Countdown state gating future behavior |
| Time Token net tally | Accumulator feeding score and door-unlock |
| Souvenir Crown kill-tracking | Per-hazard counter, deterministic under replay |
| Recent death counter (mercy rule) | Raw fact — the "is mercy rule active" boolean is **derived**, never stored |

### Not in GameState

| Item | Why excluded |
|---|---|
| Pre-rolled schedule | Pure function of seed — lives in Generation Output |
| 4 non-cosmetic RNG cursors | Consumed at generation time (ADR-0005), don't exist at runtime |
| Cosmetic RNG cursor | Client-only, never affects score/death, would cause false replay mismatches |
| `mercyRuleActive` boolean | Derived: `recentDeathCounter >= THRESHOLD` — storing it creates two sources of truth |
| Monument history (separate array) | Redundant — monuments are entities in the entity array; count is derived from `entities.filter(e => e.type === 'monument').length` |

### Client-side state (not in simulation)

| State | Owner | Notes |
|---|---|---|
| Camera, tweens, particle emitters, screen shake | `client` (Phaser scenes) | Purely presentational, rebuilt from nothing on scene restart |
| UI state (HUD values, menu state) | `UIScene` | Reads from GameState each frame, never writes to it |
| Persistent player data (best stats, unlocks) | `SaveManager` (client, localStorage) | Populated from GameState at run-end, not read by simulation |

### Practical shape

```typescript
// Simulation version — bumped manually on any determinism-affecting change
// (schedule generation, physics order, RNG stream usage, entity behavior).
// Non-determinism-affecting changes (event descriptions, comments) don't require a bump.
export const SIM_VERSION = '1.0.0';

// --- Entity types ---

// Exhaustiveness guard — compile error if a switch misses a union arm
function assertNever(x: never): never {
  throw new Error(`Unhandled entity variant: ${JSON.stringify(x)}`);
}

// Shared spatial fields — all entities have these, EntityView only needs these
interface EntityBase {
  id: string;           // String(counter) from GameState.nextEntityId, stable for lifetime
  spawnTick: number;    // diagnostic — state.tick at spawn time, not encoded into id
  x: number; y: number;
  vx: number; vy: number;
}

// Behavioral categories matching GDD object catalog groupings
type EntityCategory = 'platform' | 'powerUp' | 'reliefItem' | 'enemy'
  | 'chaoticSpawn' | 'hazard' | 'collectible' | 'misc';

// Type-specific fields — discriminated by both `category` and `type`
// Full 30+ arm union lives in simulation/types.ts, sourced from GDD catalog
type EntityData =
  | { type: 'player'; category: 'misc'; state: PlayerState }
  | { type: 'buoyPlatform'; category: 'platform'; floatHeight: number }
  | { type: 'grudgePlank'; category: 'platform'; hitHard: boolean }
  | { type: 'magpie'; category: 'enemy'; targetId: string | null; behaviorState: MagpieBehavior }
  | { type: 'mysteryCrate'; category: 'chaoticSpawn'; contents: ScheduleEntry }
  | { type: 'monument'; category: 'hazard' }
  // ... remaining ~25 arms from GDD §4-5, each tagged with category + type
  ;

// Player-specific state — nested inside the player entity, not flat on GameState.
// All fields have memory that gates future tick behavior.
interface PlayerState {
  grounded: boolean;           // updated by collision resolution each tick
  prevJumpHeld: boolean;       // for press-edge detection (!prev && current)
  coyoteTimer: number;         // ticks remaining where jump is legal after leaving ground
                               // 0.09s × 60 ticks/s ≈ 5-6 ticks — round direction is a gameplay-feel choice, document which
  jumpBufferTimer: number;     // ticks remaining where landing will trigger a buffered jump
                               // 0.10s × 60 ticks/s = 6 ticks
  // ... other player-specific fields (active power-up effects, etc.)
}

type Entity = EntityBase & EntityData;

// Centralized entity creation — all spawns go through this helper.
// Never increment nextEntityId or construct entities ad hoc at call sites.
function spawnEntity(state: GameState, data: EntityData, events?: SimEvent[]): Entity {
  const entity: Entity = {
    id: String(state.nextEntityId++),
    spawnTick: state.tick,
    x: 0, y: 0, vx: 0, vy: 0,
    ...data,
  };
  state.entities.push(entity);
  events?.push({ type: 'entitySpawned', entityId: entity.id, entityType: data.type });
  return entity;
}

// Centralized entity removal — deferred to end-of-tick, never mid-iteration.
// Mirrors Unity's Destroy() / Bevy's Commands::despawn() pattern:
// mark now, compact once after all system passes complete.
function removeEntity(
  id: string,
  pendingRemovals: Set<string>,
  events?: SimEvent[],
  cause?: DespawnCause
): void {
  pendingRemovals.add(id);
  events?.push({ type: 'entityDespawned', entityId: id, cause: cause ?? 'expired' });
}

// Reference-validity check — consults pendingRemovals so same-tick removals
// are visible to later system passes within the same step() call.
function isAlive(state: GameState, pendingRemovals: Set<string>, id: string): boolean {
  if (pendingRemovals.has(id)) return false;
  return state.entities.some(e => e.id === id);
}

// Note: monotonic, never-reused IDs (from nextEntityId counter) guarantee
// a stale reference can only resolve to "not found" — never to an unrelated
// newer entity that reused the same ID. This prevents silent retargeting bugs
// (e.g. Magpie latching onto a wrong entity after its real target despawned).

// System functions narrow by category first, then type:
// function updateHazards(entities: Entity[], state: GameState) {
//   for (const e of entities) {
//     if (e.category !== 'hazard') continue;
//     // e is narrowed to hazard sub-union (~12 arms, not all 30+)
//     switch (e.type) {
//       case 'monument': /* ... */ break;
//       default: assertNever(e);
//     }
//   }
// }

// --- Entity lifecycle within step() ---

// Deferred removal: system functions call removeEntity() during their update,
// which adds to a tick-scoped pendingRemovals set (never stored in GameState).
// Single compaction pass at the end of step() applies all removals at once.
// This prevents mid-iteration index-shifting bugs when multiple entities are
// removed in the same tick by different system passes.
//
// function step(state, run, input, dt, events?) {
//   const pendingRemovals = new Set<string>();  // tick-scoped, discarded after
//   updatePlayer(state, input, pendingRemovals, events);
//   updateHazards(state, pendingRemovals, events);
//   updateEnemies(state, pendingRemovals, events);
//   // ... all system passes share the same pendingRemovals
//
//   // Single compaction point — only place entities[] shrinks by removal
//   if (pendingRemovals.size > 0) {
//     state.entities = state.entities.filter(e => !pendingRemovals.has(e.id));
//   }
// }

// Stale-reference resolution: entity types that hold cross-entity references
// (Magpie.targetId, Copycat.mirrorId, Fixer's repair target) self-heal on read
// using isAlive(). No reverse-reference index needed at this entity count (~3 types).
//
// function updateMagpie(magpie, state, pendingRemovals) {
//   if (magpie.targetId && !isAlive(state, pendingRemovals, magpie.targetId)) {
//     magpie.targetId = pickNewTarget(state, pendingRemovals);
//   }
// }

// --- Session + State types ---

// Generation Output — built once from seed, never mutated, never hashed
interface GeneratedRun {
  seed: string;
  simVersion: string;   // stamped at generation time from SIM_VERSION
  schedule: ScheduleEntry[];
}

// Session — pairs GeneratedRun with its corresponding GameState
// Constructed via createSession(seed), the single entry point for both client and server
interface Session {
  state: GameState;
  run: GeneratedRun;
}

function createSession(seed: string): Session {
  const run: GeneratedRun = {
    seed,
    simVersion: SIM_VERSION,
    schedule: generateSchedule(seed),
  };
  const state = createInitialState(run); // places fixed landmarks, sets scheduleCursor: 0
  return { state, run };
}

// GameState — canonical, mutated every tick, this is what golden-replay hashes
interface GameState {
  tick: number;
  nextEntityId: number;               // monotonic counter for deterministic ID generation
  entities: Entity[];
  scheduleCursor: number;
  phase: PhaseState;
  score: number;
  deaths: number;
  antiIdleWindow: number[];           // fixed-size ring buffer
  deathLog: DeathEvent[];             // capped to what mechanics consume
  avoidanceLog: AvoidanceEvent[];     // capped to what mechanics consume
  coffeeBreak: { active: boolean; ticksRemaining: number };
  powerUpTimers: Record<string, number>;
  timeTokenTally: number;
  souvenirCrownKills: Record<string, number>;
  recentDeathCounter: number;         // raw fact, not the derived boolean
}

// Derived — pure functions, never stored, never hashed
function isMercyRuleActive(state: GameState): boolean {
  return state.recentDeathCounter >= MERCY_THRESHOLD;
}
```

Golden-replay tests hash `GameState` only — not the schedule, not the cosmetic RNG cursor, not any derived boolean.

## Simulation event channel (ADR-0009)

`step()` accepts an optional caller-owned `SimEvent[]` array. Events are emitted at the exact mutation site inside `step()` (e.g. right after setting `entity.state = 'dead'`, push `{ type: 'death', ... }`). Events are ephemeral — write-only outputs of a single tick, read once by the client for view sync and cosmetic triggers, then discarded. Never stored in `GameState`, never hashed, never replayed, never fed back as input.

If a future feature seems to need "did event X happen recently," that's a signal it should become a counter or flag in `GameState` — not a reason to persist the event log.

### SimEvent discriminated union

```typescript
type SimEvent =
  | { type: 'entitySpawned'; entityId: string; entityType: EntityType }
  | { type: 'entityDespawned'; entityId: string; cause: DespawnCause }
  | { type: 'death'; entityId: string; killedBy: string }
  | { type: 'respawn'; entityId: string }
  | { type: 'coffeeBreakStart' }
  | { type: 'coffeeBreakEnd' }
  | { type: 'monumentErected'; entityId: string; position: { x: number; y: number } }
  | { type: 'monumentVetoed'; position: { x: number; y: number } }
  | { type: 'complaintFormActivated'; hazardType: HazardType }
  | { type: 'mercyRuleTriggered' }
  | { type: 'powerUpCollected'; entityId: string; powerUpType: PowerUpType }
  | { type: 'powerUpExpired'; powerUpType: PowerUpType }
  | { type: 'souvenirCrownSizeUp'; hazardEntityId: string }
  | { type: 'timeTokenCollected'; delta: number }
  | { type: 'bitScatter'; position: { x: number; y: number } }
  | { type: 'doorUnlocked' }
  | { type: 'phaseChanged'; phase: Phase };
```

Client-side consumption is an exhaustive `switch` — the compiler catches missing cases when new event types are added.

### step() signature

```typescript
function step(
  state: GameState,
  run: GeneratedRun,
  input: Input,
  dt: number,
  events?: SimEvent[]  // caller-owned, push-only; client passes one, server omits
): void;
```

This is the standard simulation/presentation event-channel split (Overwatch GDC 2017, Unity Animation Events, Unreal GAS notify events) — not event-sourcing/CQRS, which is a storage pattern.

### step() canonical ordering (named rule)

The system-function call order inside `step()` is fixed and documented — same discipline as the physics integration order and jump update order. A wrong order doesn't crash; it produces subtly different simulation results that only manifest as replay verification failures or one-frame-off behavioral glitches.

```typescript
function step(state: GameState, run: GeneratedRun, input: Input, dt: number, events?: SimEvent[]) {
  const pendingRemovals = new Set<string>();

  // 1. Materialize this tick's scheduled spawns (platforms, hazards, power-ups, etc.)
  //    Must precede movement/collision so new platforms exist before player resolves against them.
  processScheduledSpawns(state, run, events);

  // 2. Decrement all timers (power-up duration, Coffee Break countdown).
  //    Before the systems that check "is this timer still active" so a timer hitting zero
  //    this tick is already reflected in this tick's behavior, not next tick's.
  //    (Coyote/jump-buffer timers handled internally within step 3, per jump-ordering rule.)
  updateTimers(state, pendingRemovals, events);

  // 3. Player: input → integrate → resolve X → resolve Y vs. platforms
  //    (per existing physics-order rule and jump-order rule)
  updatePlayer(state, input, pendingRemovals, events);

  // 4. Enemies/moving hazards: use this tick's resolved player position for targeting/chasing.
  //    Same "move before collision" treatment as the player.
  //    Stale-reference self-heal via isAlive() (deferred-removal pattern) happens here.
  updateEnemies(state, pendingRemovals, events);

  // 5. Player vs. hazard collision — uses resolved positions from steps 3 & 4.
  //    Death events appended to deathLog here.
  checkHazardCollisions(state, pendingRemovals, events);

  // 6. Player vs. collectible collision (power-ups, relief items, bits) — same position-dependency,
  //    separate outcome type from hazard collision.
  checkCollectibleCollisions(state, pendingRemovals, events);

  // 7. Anti-idle window update, using this tick's resolved player position.
  //    If threshold crossed: run solvability veto, spawn Monument if it passes.
  //    Placed AFTER hazard collision (step 5) so a freshly-spawned Monument does NOT
  //    retroactively hit the player the same tick it appears — one-tick grace window.
  //    (Pending GDD confirmation on intended feel; defaulting to grace.)
  updateAntiIdle(state, pendingRemovals, events);

  // 8. Score/tally updates, derived from this tick's collision/pickup events.
  //    Time Token tally, Souvenir Crown per-hazard counts, etc.
  updateScoreAndTallies(state, events);

  // 9. Phase/clock progression.
  updatePhase(state, run, events);

  // 10. Single deferred-removal compaction — always last.
  if (pendingRemovals.size > 0) {
    state.entities = state.entities.filter(e => !pendingRemovals.has(e.id));
  }

  state.tick++;
}
```

**Log-read-timing rule:** Systems that read death/avoidance logs to gate their own behavior (Complaint Form, Regifter, Understudy) must read the log as of **tick-start** — this tick's own collision-appended entries are not visible to them until the next tick. This prevents same-tick-reaction hazards identical to the Monument case. Structurally, this is already achieved by the ordering above: log-reading behavior in `updateEnemies` (step 4) runs before collision checks that append to the logs (steps 5–6).

**Same-tick Monument lethality (open design question):** The default ordering gives a one-tick grace window — the Monument spawns in step 7, after hazard collision ran in step 5, so the player has one tick to react before collision fires against it next tick. This matches the GDD's framing of Monuments as warnings/consequences ("a statue of you, where you stood") rather than instant unavoidable damage. Pending explicit GDD confirmation; if the GDD intends instant punishment, move Monument-spawn before step 5.

## Physics (deterministic core)

**Determinism prohibition:** No code in `simulation/` may call `crypto.randomUUID()`, `Math.random()`, `Date.now()`, `performance.now()`, or any other source of non-deterministic input. All randomness comes from the seeded PRNG (SFC32 sub-streams). All identity generation comes from `GameState.nextEntityId`. All time comes from `GameState.tick`. This is a restatement of the determinism contract from the `step()` acceptance criteria, made explicit here because ID generation and timing are the most likely places a contributor would instinctively reach for a non-deterministic source.

- Custom fixed-point-safe or carefully-ordered floating point AABB physics in `simulation/`:
  - Gravity/velocity integration
  - Static collision (player vs. platform layout, pre-rolled at generation per ADR-0005)
  - Dynamic collision (player vs. hazard/collectible — pre-rolled placement, runtime AABB test)
- **Order of operations is fixed and documented** (integrate velocity → resolve X → resolve Y → check triggers) since float operation order affects determinism across environments
- No trig/transcendental functions in the hot path unless proven bit-identical across V8 on client and server (same engine, so low risk, but flagged as a determinism risk area regardless)

### Jump update ordering (named rule)

The player jump system has a fixed three-step order, same discipline as the physics integration order above:

1. **Update timers** based on current tick's ground state (before this tick's jump attempt)
   - `coyoteTimer`: reset to max if grounded, else decrement toward 0
   - `jumpBufferTimer`: reset to max on press edge (`input.jump && !prevJumpHeld`), else decrement toward 0
2. **Attempt jump**: fire if `(pressEdge || jumpBufferTimer > 0) && (coyoteTimer > 0 || grounded)` — then clear both timers
3. **Update `prevJumpHeld`** last, reflecting *this* tick's input for next tick's edge detection

```typescript
function updatePlayerJump(player: Entity & { type: 'player' }, input: Input) {
  const ps = player.state;
  const jumpPressedEdge = input.jump && !ps.prevJumpHeld;

  // 1. Timers
  ps.coyoteTimer = ps.grounded ? COYOTE_TIME_TICKS : Math.max(0, ps.coyoteTimer - 1);
  if (jumpPressedEdge) ps.jumpBufferTimer = JUMP_BUFFER_TICKS;
  else ps.jumpBufferTimer = Math.max(0, ps.jumpBufferTimer - 1);

  // 2. Jump attempt
  const canJump = ps.coyoteTimer > 0 || ps.grounded;
  if ((jumpPressedEdge || ps.jumpBufferTimer > 0) && canJump) {
    player.vy = JUMP_VELOCITY; // -624 px/s per GDD
    ps.coyoteTimer = 0;
    ps.jumpBufferTimer = 0;
  }

  // 3. Edge flag — last
  ps.prevJumpHeld = input.jump;
}
```

## Solvability Veto

A single shared pure function `checkSolvability()` in `simulation/` validates that a path exists from spawn pocket to SHIP IT door given all currently committed permanent entities. Called from two sites:

### Generation-time veto (inside `generateSchedule()`)

For every permanent scheduled spawn (pit, static spike cluster), run `checkSolvability()` against the nav grid with all previously committed permanent spawns in place. Blocked → relocate to nearest legal position. Sequential — spawn N's legality depends on spawns 1 through N-1.

### Runtime veto (inside `step()`, Monument creation)

Monuments are placed at the player's current position at runtime (anti-idle rule), not pre-rollable. Before committing, run the **same** `checkSolvability()` against current `state.entities` filtered to permanent types, plus the candidate monument. If blocked → **veto the monument entirely** (do not relocate — a moved statue defeats the narrative purpose). The anti-idle rule's other consequences (knockback, pigeon) still apply; only the physical hazard entity is withheld.

The full flood-fill is affordable at this call site because Monument creation is a rare event gated by the anti-idle cooldown (a handful of times per 5-minute run at most), not a per-tick operation. The nav grid is small (~960×540 world space discretized to jump-arc cells ≈ a few dozen total nodes). Sub-millisecond cost.

**Note:** "runs inside the deterministic simulation" is a **correctness** constraint (same-input-same-output), not a performance constraint. Client and server may take different wall-clock time to execute the same pure function and still produce bit-identical `GameState` — that's the premise of replay verification.

### Implementation shape

```typescript
// simulation/solvability.ts — single implementation, both call sites
function checkSolvability(
  permanentEntities: Entity[],
  from: Point,
  to: Point,
  jumpArc: JumpArcParams
): boolean {
  const grid = buildNavGrid(permanentEntities, jumpArc); // derived, rebuilt each call
  return floodFillReachable(grid, from, to);
}

// Runtime Monument veto inside step()
function trySpawnMonument(
  state: GameState,
  playerPos: Point,
  events?: SimEvent[]
): void {
  const permanentEntities = state.entities.filter(isPermanentType);
  const candidate = { x: playerPos.x, y: playerPos.y, type: 'monument' as const, category: 'hazard' as const };

  if (!checkSolvability(
    [...permanentEntities, candidate as Entity],
    SPAWN_POCKET,
    SHIP_IT_DOOR,
    JUMP_ARC
  )) {
    events?.push({ type: 'monumentVetoed', position: playerPos });
    return; // veto — no relocation, anti-idle knockback/pigeon still apply
  }

  spawnEntity(state, candidate, events);
}
```

The nav grid is **derived** — rebuilt from `state.entities` on each call, never cached or stored in `GameState` (consistent with ADR-0008: fully reconstructable from canonical state).

## Rendering pipeline

1. Boot: generate all procedural textures once via `Graphics.generateTexture()`, cache by key
2. Each render frame (not simulation tick): `EntityView` layer reads latest `GameState`, positions sprites
3. `GameScene` renders world; `UIScene` renders HUD independently, both driven by the same `GameState` reference
4. Cosmetic variation (color/shape variants) resolved once at entity-spawn time from the cosmetic PRNG stream, cached on the entity's view — not re-rolled per frame

## Scene interaction

| Scene | Pausable | Reads | Writes |
|---|---|---|---|
| `GameScene` | Yes (ad breaks, portal pause) | `GameState` | Drives `simulation.step()` |
| `UIScene` | No — stays active during pause | `GameState` (read-only) | None |

Pause = freeze `simulation.step()` calls + `GameScene.scene.pause()` + `AudioContext.suspend()`, all triggered from one portal-adapter callback so they can't drift out of sync.

## Data flow summary

```
Input (keyboard/touch) → InputAbstraction → { left, right, jump }
        ↓
simulation.step(state, run, input, dt, events?) → void (mutates state, pushes events)
        ↓                                  ↓
  client: reads events[] for             server: replay verifies
  view sync + cosmetic triggers          final state matches submitted score
  → Phaser renders                       (events collector omitted)
```

## Server replay verification loop

- Client submits `{ seed, simVersion, inputLog, claimedScore }`
- Server checks `simVersion` against its current `SIM_VERSION` — mismatch → reject with `'version-mismatch'`, do not attempt replay (a run played under version N cannot be verified by version N+1's logic)
- On match: server calls `createSession(seed)`, re-runs `simulation.step()` through `inputLog` headlessly (no Phaser, no rendering, no events collector)
- Final `GameState.score` must match `claimedScore` bit-for-bit before Redis/Postgres write
- Any score mismatch → reject submission, no partial credit
- Multi-version replay support (keeping old `step()` logic alive to verify stale submissions) is deferred — reject-and-log is the v1 scope; only add version-dispatch complexity if analytics show meaningful submissions land in the deploy-gap window

## Acceptance criteria

- [ ] `simulation.step()` is deterministic: same `(state, input)` in → same mutations → same resulting `state`, no reads from or writes to external scope. Mutates `state` in place — callers must not assume the input survives unchanged. Golden-replay tests `structuredClone(state)` before snapshot ticks.
- [ ] No Phaser Arcade Physics body ever influences a value read by scoring/death logic
- [ ] `EntityView` creation/destruction is driven by `SimEvent` emissions (primary) with ID-based diff as bootstrap on scene creation (secondary) — no manual sprite management in gameplay code
- [ ] Server replay loop produces identical final `GameState` hash to client for all golden-replay fixtures
- [ ] Pause freezes simulation stepping, not just rendering — verified independently per subsystem: (a) no `simulation.step()` calls fire, (b) no Arcade Physics body position changes, (c) no `GameScene`-scoped Tween advances, (d) `AudioContext` is suspended. Each checked separately against pinned Phaser 4.2.0+, re-verified on version bumps. `UIScene` tweens/animations must remain active during `GameScene` pause.
- [ ] No boolean/enum flag exists in `GameState` that is a pure function of other `GameState` fields — every derived value has a co-located pure derivation function
- [ ] `SimEvent` emissions are never stored in `GameState`, never included in replay hash, never fed back as input to `step()`
- [ ] Every mutation site in `step()` that spawns, despawns, or changes entity state emits a corresponding `SimEvent` — verified by bootstrap-diff reconciliation catching orphaned views in tests
- [ ] `createSession(seed)` is the single entry point for session initialization on both client and server — no separate `createInitialState()` without a `GeneratedRun`
- [ ] Server replay verification rejects with `'version-mismatch'` when submitted `simVersion` differs from current `SIM_VERSION` — never silently attempts cross-version replay
- [ ] Client accumulator caps at `MAX_TICKS_PER_FRAME` and drops remaining backlog — verified by backgrounding the browser tab for 10+ seconds mid-run and confirming the game resumes normally without fast-forward or spiral-of-death
- [ ] Client processes `SimEvent[]` in strict array order with no reordering or deduplication
- [ ] `checkSolvability()` is a single shared pure function in `simulation/` used by both `generateSchedule()` and the runtime Monument-spawn system — no duplicated flood-fill logic
- [ ] Monument creation is vetoed (not relocated) when `checkSolvability()` fails against current permanent entities — golden-replay fixture exercises two Monument triggers at positions that jointly-but-not-individually brick a corridor
- [ ] Input is sampled once per frame before the accumulator loop — all ticks within the same frame receive the same `Input` snapshot
- [ ] `jump` is a held-state boolean; press-edge detection and coyote/jump-buffer timers live in `PlayerState`, not in the input abstraction
- [ ] Jump update order is: timers → jump attempt → `prevJumpHeld` update (last) — golden-replay fixtures exercise both a buffered jump (press before landing) and a coyote jump (press after leaving platform)
- [ ] Input log has exactly one `Input` entry per simulation tick — length equals total ticks in the run
- [ ] Entity removal is deferred via `pendingRemovals: Set<string>` scoped to each `step()` call — no mid-iteration splicing of `entities[]`. Single `filter()` compaction at end of `step()`
- [ ] All cross-entity reference checks (Magpie, Copycat, Fixer) use `isAlive()` which consults `pendingRemovals` — golden-replay fixture exercises two entities removed in the same tick where one is another's active target
- [ ] `step()` system-function call order matches the canonical 10-step sequence — no system function is reordered without updating the documented sequence and reviewing downstream order-dependencies
- [ ] Freshly-spawned Monument does not trigger hazard collision against the player in the same tick it appears (one-tick grace) — golden-replay fixture covers the idle-threshold-crossed-while-standing-on-spawn-point case
- [ ] Log-reading systems (Complaint Form, Regifter, Understudy) do not see death/avoidance log entries appended in the same tick — verified by fixture where a death and a Complaint Form activation occur on the same tick

## Dependencies

- [docs/adr/0004-monorepo-shared-simulation.md](../../docs/adr/0004-monorepo-shared-simulation.md)
- [docs/adr/0005-pre-roll-all-outcomes.md](../../docs/adr/0005-pre-roll-all-outcomes.md)
- [docs/adr/0006-replay-verification.md](../../docs/adr/0006-replay-verification.md)
- [docs/adr/0008-canonical-state-vs-generation-output-vs-derived.md](../../docs/adr/0008-canonical-state-vs-generation-output-vs-derived.md)
- [docs/adr/0009-ephemeral-event-channel.md](../../docs/adr/0009-ephemeral-event-channel.md)
- [docs/adr/0007-arcade-physics-cosmetic-only.md](../../docs/adr/0007-arcade-physics-cosmetic-only.md)
