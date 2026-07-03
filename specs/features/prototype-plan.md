# Prototype Plan

**Path:** features/prototype-plan
**Scope:** What the first buildable milestone is — feature scope, architecture validation targets, "done" definition, and explicit exclusions. Excludes backend/server work, portal integration, and the full P1–P3 roadmap.

## Approach: rebuild, not port

Rebuild from scratch on the new architecture (simulation core, Phaser 4, monorepo) rather than porting P0. P0's input handling, entity model, and rendering approach all predate the coyote-timer, discriminated-union, and event-channel decisions made across the architecture and schedule-generation sessions. A port would mean reworking most of the code anyway.

## Feature scope

### P0 parity (the foundation)

Everything the shipped prototype already does, rebuilt on the specified architecture:

| Feature | Architecture dependency |
|---|---|
| Room (960×540 single-screen arena) | `Phaser.Scale.FIT`, `GameScene` |
| Clock (0:00 → 5:00, real-time) | `GameState.tick`, `FIXED_DT` |
| SHIP IT door (unlocks at 5:00) | Schedule cursor, `doorUnlocked` SimEvent |
| 6 hazard types (static spikes, etc.) | `EntityBase` + `EntityData` discriminated union, `assertNever` exhaustiveness |
| Deterministic scheduler | `generateSchedule(seed)`, `ScheduleEntry[]`, pre-rolled outcomes (ADR-0005) |
| Telegraph warnings (0.7s gold blink) | `telegraphTicks` on `ScheduleEntry`, telegraph-cap enforcement |
| Spawn pocket (left wall safe zone) | Placement pipeline exclusion zone |
| Death / instant respawn | `SimEvent { type: 'death' }` + `{ type: 'respawn' }`, `PlayerState` |
| HUD (timer, score, deaths) | `UIScene` reading `GameState` per frame |
| Title + win screens | Screen flow (Title → Gameplay → End-of-Run → Title) |
| Mobile controls | New `InputAbstraction` with split-screen zones + dedicated jump button |
| Vignette effect | Cosmetic, client-only |

### P1 additions (architecture validation)

Two P1 systems chosen to prove architecture paths that P0 parity alone would leave unvalidated:

| System | What it validates | Why this, not other P1 items |
|---|---|---|
| **Monument + solvability veto** | Runtime `checkSolvability()` — the same flood-fill code that schedule generation uses at generation time. Exercises: anti-idle detection → candidate Monument → solvability check → veto-or-commit → `SimEvent` emission → view creation. The cross-cutting piece both sessions depend on. | Bus Stops, Coffee Breaks, Bits/Piggy Bank, Magpie, Mystery Crate are all simpler entity types that don't exercise novel architecture paths. |
| **Time Tokens (±10s)** | Audio-as-mechanic — the ticking-direction honest tell is load-bearing (GDD's Honesty Split), not decorative. Exercises: Web Audio synthesis (ZzFX-style), frequency-band routing, a mechanical signal that players must parse by ear. Also validates `timeTokenTally` in GameState and the 4:00 floor cap. | Without Time Tokens, the prototype's audio would be pure SFX polish — no mechanic would depend on audio being correct. |

### Deferred P1 items (post-prototype)

| Item | Why deferred |
|---|---|
| Bus Stops | Simple entity type, no novel architecture path |
| Coffee Breaks | Global event type, straightforward timer — no hidden complexity |
| Bits + Piggy Bank + death scatter | Collectible economy, valuable but doesn't prove architecture |
| Magpie | Enemy AI, interesting but self-contained |
| Mystery Crate | Chaos-spawn resolution, already specified in schedule-generation spec |

### Deferred entity behavior specs (needs spec before implementation)

These entities have their architecture integration points specified (spawn category, log-read-timing rule, step ordering) but their gameplay behavior specs are deferred. Each needs a per-entity spec file before entering implementation:

| Entity | Integration specified in | What's missing |
|---|---|---|
| Regifter | schedule-generation.md (runtime-resolved spawn, tieBreakDraw) | What the spawned entity does — lookup against avoidance log, manifestation behavior |
| Understudy | schedule-generation.md (runtime-resolved spawn) | 20s observation phase, transformation into most-killed hazard type |
| Complaint Form | schedule-generation.md (runtime-resolved spawn) | Recall mechanic — deactivate last-killer hazard type for 20s |
| Intern's Broom | architecture.md (despawn cause in SimEvent) | Sweep mechanic — which lane, how many entities cleared, single-use |
| Fixer | GDD §4.6 only | Re-arm/repair behavior, indirect-kill-only interaction model |

## Audio scope (in prototype)

Full Web Audio implementation, not a "layer on later" concern:

| Component | Scope |
|---|---|
| One-shot SFX | ZzFX-style parameter arrays → AudioBuffer. Death, respawn, pickup, spawn telegraph. |
| Sustained sounds | OscillatorNode/GainNode graphs. Time Token ticking (direction = honest tell). |
| Frequency-band routing | Per-category GainNode → BiquadFilterNode → master bus. Hazard = low, value = high, relief = mid. |
| Master volume control | Settings screen slider → master bus gain. |
| Pause integration | `AudioContext.suspend()`/`resume()` tied to GameScene pause. |

## Server / backend scope

**None.** Pure client-side. No Fastify, no PostgreSQL, no Redis, no real replay verification.

### Dev-only determinism self-check

The client-side pieces that feed future server verification are included at zero extra cost:

- `createSession(seed)` — already required
- `simVersion` stamping — already required
- Input-log recording — already required (flat array, one `Input` per tick)

With these in place, add a **dev-only self-check**: after a run completes, replay the recorded input log against a freshly created session client-side and diff the final `GameState` against what was actually played. This validates determinism without needing a server.

```typescript
// Dev-only, behind a flag — not shipped to players
function selfCheckReplay(seed: string, inputLog: Input[], expectedFinalState: GameState): boolean {
  const session = createSession(seed);
  for (const input of inputLog) {
    step(session.state, session.run, input, FIXED_DT);
  }
  return deepEqual(session.state, expectedFinalState);
}
```

This is a poor-man's version of what the server would eventually do — cheap insurance that determinism holds before a backend exists to catch violations.

## Mobile controls

Fresh implementation against the new `{ left, right, jump }` input abstraction and per-frame-sampling contract (architecture spec, Answer 9). P0's touch-zone layout (split-screen halves, dedicated jump button carved from right zone) is reused as a UX reference — the feel/placement decisions don't need re-litigating. The wiring underneath targets the new `Input` interface and `PlayerState` coyote/jump-buffer fields, which didn't exist in P0.

## "Done" definition

### Floor: runs without crashing

A playable browser build that completes a full 5-minute run with the specified scope (P0 parity + Monument/solvability + Time Tokens), audio functioning, desktop + mobile input working.

### Actual bar: answers three qualitative questions

The prototype exists to prove the core loop is fun before building 30+ entity types. "Done" means the build exists **and** you have real answers (from playing it, not from reasoning) to:

1. **Does the density ramp feel tense without feeling unfair?** — Directly tests the ramp/no-jitter decisions from the schedule-generation session. If the answer is "no," the ramped-interval parameters or phase targets need tuning before building more content on top.

2. **Does coyote-time/jump-buffer timing feel right?** — The architecture session caught a real implementation gap here (0.09s × 60 ≈ 5.4 ticks — round direction is a gameplay-feel choice). This is where the rounding decision gets made by feel, not by theory.

3. **Is the telegraph warning readable and fair at Last Words density?** — The telegraph-cap finding from schedule generation (≤2 concurrent telegraphs, delay-forward enforcement) was derived from math. This is where it gets validated against actual visual noise at 100+ objects.

Golden-replay test correctness is **not** part of "done" — that comes when the dev-only self-check is promoted to a real test harness, which is a separate work item.

## Acceptance criteria

- [ ] Monorepo structure matches tech-stack spec: `packages/simulation/`, `packages/client/`, `packages/server/` (server empty but present)
- [ ] `simulation/` has zero Phaser/DOM/Node imports — enforced by tsconfig path restrictions
- [ ] Full 5-minute run is completable: spawn, survive, SHIP IT door unlocks, touch to win
- [ ] Monument spawns on idle (5s horizontal threshold), solvability veto blocks if path is bricked
- [ ] Time Tokens collected: −10s advances clock, +10s delays clock, 4:00 floor enforced, ticking audio direction is correct
- [ ] Audio functions: SFX on death/respawn/pickup, frequency-band routing audible, master volume slider works
- [ ] Desktop input (keyboard) and mobile input (touch zones) both produce correct `{ left, right, jump }` with coyote time and jump buffer
- [ ] Dev-only determinism self-check passes for at least one full run
- [ ] Qualitative questions (density feel, jump timing, telegraph readability) have been answered by actually playing the build

## Dependencies

- [specs/features/architecture.md](architecture.md) — all of it
- [specs/features/schedule-generation.md](schedule-generation.md) — `generateSchedule()`, phase targets, telegraph cap
- [specs/features/tech-stack.md](tech-stack.md) — Phaser 4, Vite, pnpm workspaces, SFC32 PRNG, Web Audio
- [specs/features/save-manager.md](save-manager.md) — persistence across prototype runs
- [specs/features/ui-screens.md](ui-screens.md) — screen flow, HUD content
