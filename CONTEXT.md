# One Second Longer — "The Reject Bin"

A single-screen 2D survival arena where spawned objects accumulate over a fixed 5-minute clock. The player survives the room's increasing density and crosses to the exit door when the clock expires.

## Language

### The Room

**Reject Bin**:
The single-screen arena where all gameplay occurs — a QA dump room that fills with spawned objects over 5 minutes.
_Avoid_: level, stage, map

**Spawn Director**:
The deterministic schedule generator that pre-rolls every spawn event (type, position, time, parameters) from a seeded PRNG before the session starts.
_Avoid_: spawner, level generator, procedural generator

**Phase**:
One of five time windows (Naive Room, The Fill, Crowding, The Flood, Last Words) that define the spawn cadence and category bias across the 5-minute session.
_Avoid_: wave, round, stage

**The Crossing**:
The post-5:00 traversal through the frozen, saturated room to reach the SHIP IT door. Spawning has ceased; the accumulated room is the final exam.
_Avoid_: final level, boss, endgame

**SHIP IT Door**:
The exit on the right wall that unlocks at 5:00 (or earlier via Time Tokens, floor 4:00). Touching it after unlock ends the session.
_Avoid_: goal, finish line, portal

**Spawn Pocket**:
The guaranteed no-spawn zone at the left wall (x < 110) where the player respawns. Safe from scheduled spawns; not exempt from the anti-idle rule.
_Avoid_: safe zone, start area

**Reserved Corridor**:
The door-approach column (x > 895) that never receives permanent spawns. Mobile hazards may transit through it.
_Avoid_: exit lane, safe path

### Objects & Categories

**Reject**:
Any object spawned into the Reject Bin. Rejects were "rejected" from a fictional game for different reasons — the premise licenses mixed-quality spawns.
_Avoid_: item, entity, thing

**Reject Variant**:
A defective version of a power-up — same silhouette, crooked label, broken behavior. The label lies; the physical tell does not.
_Avoid_: cursed item, trap item

**Monument**:
A permanent spike-hazard in the shape of the player, erected where they stood idle for 5+ seconds. Punishes the spot, not the player.
_Avoid_: penalty, idle punishment

**Coffee Break**:
A scheduled 10-second pause at fixed times (~1:30, ~3:00) where spawning stops and moving hazards freeze. The only free idle window. Static hazards remain active.
_Avoid_: safe period, timeout

**Time Token**:
A spawned token that edits the master clock (−10s or +10s). The ticking direction is the honest tell; the label may lie.
_Avoid_: clock pickup, timer item

### Rules & Systems

**Honesty Split**:
The readability contract: mechanical channels (color, shape, motion, telegraph) always tell the truth; linguistic channels (labels, names, signs, arrows) are allowed to lie.
_Avoid_: trust system, visual language

**Solvability Veto**:
A nav-grid flood-fill reachability check (using the real jump arc) from spawn pocket to door, run before any permanent spawn commits. Blocked spawns relocate.
_Avoid_: path check, accessibility check

**Daily Bin**:
A date-seeded session mode where all players worldwide get the same room for 24 hours, with a shared leaderboard.
_Avoid_: daily challenge, daily run

**Simulation Core**:
The pure-TypeScript game logic module (spawn director, physics stepping, collision) that runs identically on the client (with rendering) and the server (headless, for replay verification). Contains no Phaser or DOM dependencies.
_Avoid_: game engine, game logic (too vague)

**Replay Verification**:
Server-side re-simulation of a player's submitted input log against the session seed to compute authoritative results. The client's claimed score is never trusted.
_Avoid_: anti-cheat, score validation

**Golden Replay**:
A recorded input log + seed pair with snapshotted output, used as a regression test for the simulation core. Any snapshot-breaking code change requires explicit review.
_Avoid_: replay test, integration test

**Generation Output**:
The fully-resolved spawn schedule and all seed-derived data, computed once from the session seed at run start. Immutable during play; regenerated identically by client and server from the same seed.
_Avoid_: level data, run config, session setup

**GameState**:
The canonical mutable state that the Simulation Core steps forward each tick — entity positions, score, deaths, active timers, and every log read by gameplay mechanics. What Replay Verification hashes. Excludes Generation Output (regenerable from seed) and any value derivable from existing GameState fields.
_Avoid_: world state, full state

**Session**:
The paired unit of a Generation Output and its corresponding GameState, constructed via `createSession(seed)`. The single entry point for session initialization on both client and server. Carries a simulation version stamp for cross-deploy replay verification safety.
_Avoid_: game instance, run object

### Visual & Audio

**Shape-Primitive Library**:
Reusable draw functions (`drawSpikeShape()`, `applyCrackOverlay()`, etc.) that generate all game visuals procedurally via Phaser's `Graphics` API. No image assets exist.
_Avoid_: sprite library, art assets

**Color-Constant Table**:
The canonical color assignments enforcing the Honesty Split: coral = lethal, mint = safe/helpful, gold = value, dull bronze = fake value.
_Avoid_: palette, color scheme

**Frequency-Band Bus**:
The Web Audio routing architecture where each object category owns a frequency band (hazards low, value high, relief mid), enforced by per-category `BiquadFilterNode` chains.
_Avoid_: audio mixer, sound system

## Relationships

- The **Spawn Director** pre-rolls a complete timeline of **Rejects** using 5 PRNG sub-streams (timing, type, placement, chaos, cosmetic) before the session starts
- Each **Reject** spawns during one of five **Phases**, which control cadence and category bias
- The **Solvability Veto** validates every permanent **Reject** placement before it commits
- **Time Tokens** edit the master clock but cannot open the **SHIP IT Door** before 4:00
- A **Monument** is created by the anti-idle rule, not by the **Spawn Director** — but its placement passes through the **Solvability Veto**
- The **Simulation Core** is shared between the Phaser client and the Fastify server — the server uses it for **Replay Verification**
- **Golden Replays** test the **Simulation Core** in isolation, with no Phaser dependency
- The **Shape-Primitive Library** and **Color-Constant Table** are the visual equivalent of the **Frequency-Band Bus** — each enforces the **Honesty Split** in its domain
- The **Spawn Director** produces the **Generation Output** once from a seed; the **Simulation Core** reads it as immutable input every tick alongside mutable **GameState**
- **Replay Verification** hashes **GameState** only — never **Generation Output** (regenerated from seed) and never derived values
- **Golden Replays** snapshot **GameState** at specific ticks; **Generation Output** is regenerated, not snapshotted

## Example dialogue

> **Dev:** "When the **Spawn Director** places a Mystery Crate, is its content decided at generation time or when the player opens it?"
> **Domain expert:** "At generation time — the **Simulation Core** pre-rolls all chaos outcomes. The crate at (x, y, t) always contains the same thing regardless of player behavior. Otherwise the **Daily Bin** leaderboard breaks: two players with the same seed would get different crate contents depending on interaction order."
> **Dev:** "What about the Regifter? It spawns whatever hazard type the player last avoided — that can't be pre-rolled."
> **Domain expert:** "Correct. The Regifter isn't a PRNG consumer — it's a deterministic lookup against runtime avoidance history. If there's a tie, the tie-break *is* pre-rolled (one reserved draw per Regifter spawn event), but the 'which type' logic is pure history query."

## Flagged ambiguities

- **"random"** — resolved: means *unknowable in advance by the player*, not *different per run*. All randomness is seeded; chaos you can study. (§2.6)
- **"mobile"** — resolved: means mobile browsers via web portals (Poki/CrazyGames), not native App Store/Play Store distribution. (Q1)
- **"simulation"** — resolved: specifically the **Simulation Core** package (pure TS, no Phaser). Not a synonym for "the game" or "gameplay." (Q12)
