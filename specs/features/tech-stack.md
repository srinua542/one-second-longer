# Tech Stack

**Path:** features/tech-stack
**Scope:** All technology choices for engine, language, tooling, architecture, and infrastructure — explicitly excludes gameplay mechanics, art direction rationale, and game design decisions

## Decisions

### Core

| Layer | Choice | Version |
|---|---|---|
| Game framework | Phaser 4 | 4.2.0+ |
| Language | TypeScript | strict mode |
| Bundler | Vite | via official `template-vite-ts` |
| Physics | Phaser Arcade Physics | built-in, no Matter.js |
| Package manager | pnpm | workspaces enabled |

### Repository structure

```
packages/
  simulation/   → pure TS, zero Phaser/DOM/Node deps
  client/       → Phaser 4 + Vite, imports simulation
  server/       → Fastify + PostgreSQL + Redis, imports simulation
```

### PRNG

| Property | Value |
|---|---|
| Algorithm | SFC32 (inlined, no dependency) |
| String-seed hashing | FNV-1a |
| Sub-streams | 4: type, placement, chaos, cosmetic (timing stream removed — spawn timing is fully deterministic from the ramped interval curve, zero RNG) |
| Consumption model | Type/placement consumed at generation time; chaos consumed at generation time for chaotic-spawn entity properties (Mystery Crate contents, Gacha Door destinations, Weather Vane effects, Dice faces, tie-breaks); cosmetic consumed at runtime only |
| Schedule model | Pre-roll all outcomes (ADR-0005) |

### Rendering

| Property | Value |
|---|---|
| Strategy | Procedural/geometric via `Graphics.generateTexture()` at boot |
| Image assets | None — permanent decision (ADR-0002) |
| Shared layer | Shape-primitive library + color-constant table (coral/mint/gold/dull-bronze) |
| Cosmetic variation | Drawn from isolated cosmetic PRNG stream |

### Audio

| Property | Value |
|---|---|
| Engine | Raw Web Audio API (ADR-0003) |
| One-shot SFX | ZzFX-style parameter arrays → AudioBuffer |
| Sustained sounds | OscillatorNode/GainNode graphs, start/stop tied to game state |
| Routing | Per-category GainNode → BiquadFilterNode → master bus |
| Frequency bands | Hazards = low, value = high, relief = mid |
| Concurrent voice cap | 16–24 simultaneous nodes |
| Pause integration | `AudioContext.suspend()`/`resume()` via portal adapter |

### Scaling & input

| Property | Value |
|---|---|
| Scale mode | `Phaser.Scale.FIT`, 960×540 base resolution |
| Orientation | Landscape enforced: rotate-prompt overlay (primary) + Screen Orientation Lock API (enhancement) |
| Scene architecture | Two parallel scenes: `GameScene` (pausable) + `UIScene` (always active) |
| Desktop input | Keyboard (arrow keys / WASD + space) |
| Mobile input | Split-screen movement zones (left half / right half, hold-based) + dedicated jump button (bottom-right, carved out of right zone) |
| Input abstraction | Both map to `{ left: boolean, right: boolean, jump: boolean }` consumed by simulation core |

### Portal SDK integration

| Property | Value |
|---|---|
| Pattern | Thin adapter: `PortalSDK` interface + `PokiAdapter`, `CrazyGamesAdapter`, `NullAdapter` |
| Ad pause mechanism | `GameScene.scene.pause()`/`resume()` + `AudioContext.suspend()`/`resume()` |
| Player identity | Pseudonymous UUID in localStorage (or portal SDK user ID if available) |

### Backend (Daily Bin leaderboard)

| Layer | Choice |
|---|---|
| API framework | Fastify (TypeScript) |
| Durable storage | PostgreSQL |
| Leaderboard cache | Redis sorted sets (`leaderboard:{date}`) |
| Score verification | Replay-based: headless re-simulation of input log (ADR-0006) |
| Composite ranking | `score = furthestPct * 1_000_000 - deaths` in Redis sorted set |
| Rate limiting | Redis, per playerId + IP, max 20 submissions/hour |

### Client persistence

| Property | Value |
|---|---|
| Storage | localStorage, single JSON blob via `SaveManager` |
| Contents | Best stats, cosmetic unlocks, Field Guide progress, player UUID |

### Testing

| Property | Value |
|---|---|
| Simulation regression | Golden-replay tests: recorded input logs + fixed seeds → deterministic state snapshots |
| Snapshot policy | Any snapshot-breaking simulation change requires explicit review |
| Scope | `simulation` package only, no Phaser dependency in test harness |

## Acceptance Criteria

- [ ] `packages/simulation/` compiles and runs in both browser (via client) and Node (via server) without modification
- [ ] `packages/simulation/` has zero imports from Phaser, DOM APIs, or Node-specific APIs — enforced by tsconfig path restrictions
- [ ] Same seed + same input log produces bit-identical simulation output on client and server
- [ ] Portal adapter pause freezes GameScene, game clock, physics, tweens, and audio simultaneously
- [ ] UIScene remains responsive during ad-break pause
- [ ] Mobile rotate-prompt appears in portrait orientation on all tested devices
- [ ] Touch controls support simultaneous move + jump via multi-touch
- [ ] All procedural textures generated at boot, not per-frame
- [ ] Audio frequency bands enforced by BiquadFilterNode routing, not just convention
- [ ] Golden-replay test suite exists with ≥3 recorded replays before P1 is considered complete

## Dependencies

- docs/adr/0001-phaser-4-over-phaser-3.md
- docs/adr/0002-procedural-visuals-no-image-assets.md
- docs/adr/0003-raw-web-audio-api.md
- docs/adr/0004-monorepo-shared-simulation.md
- docs/adr/0005-pre-roll-all-outcomes.md
- docs/adr/0006-replay-verification.md
