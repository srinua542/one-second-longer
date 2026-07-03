# UI Screens & HUD

**Path:** features/ui-screens
**Scope:** Screen inventory, navigation flow, HUD content and layout, end-of-run summary, Field Guide screen structure. Excludes entity rendering detail (see architecture.md), audio design, and portal-specific screens (leaderboard viewing — deferred until backend exists).

## Screen flow

```
Title Screen
    ↓
Mode Select (Classic / Daily Bin)
    ↓
Gameplay (GameScene + UIScene)
    ↓  (SHIP IT door touched or quit)
End-of-Run Summary
    ↓
Title Screen
```

### Overlay screens (not full navigation transitions)

| Overlay | Trigger | Behavior |
|---|---|---|
| Pause | Escape / pause button / tab-backgrounding | Freezes simulation (`step()` stops), `GameScene.scene.pause()`, `AudioContext.suspend()`. UIScene stays active (pause menu renders in UIScene). |
| Settings | Accessed from Pause overlay or Title screen | Audio volume slider (master bus gain). Controls reminder (read-only). |

### Screen responsibilities

| Screen | Owns | Reads |
|---|---|---|
| **Title** | Mode-select buttons, settings access, Field Guide access | `SaveData` (show streak/stats if desired) |
| **Mode Select** | Classic / Daily Bin choice. Daily Bin shows today's seed string. | Current date (for seed display) |
| **Gameplay** | `GameScene` (world rendering, simulation driving) + `UIScene` (HUD) | `GameState`, `SimEvent[]` |
| **End-of-Run Summary** | Full stat display, share-card data, "play again" / "menu" buttons | Final `GameState` snapshot |
| **Pause overlay** | Resume / Settings / Quit-to-Menu buttons | None (pure UI) |
| **Settings** | Audio volume slider, controls reminder | `SaveData` (persisted volume preference) |
| **Field Guide** | Per-type near-miss collection grid | `SaveData.fieldGuide` |

## HUD content (UIScene, during gameplay)

Minimal — the single-screen arena must stay uncluttered. Four elements, always visible:

| Element | Source | Position (guideline) | Notes |
|---|---|---|---|
| **Timer** | `state.tick` → formatted as M:SS | Top-center | Core tension driver. Counts up toward 5:00. |
| **Bits banked** | `state.score` | Top-right area | Primary feedback loop. |
| **Death count** | `state.deaths` | Small, corner (top-left or bottom-left) | Non-intrusive — information, not pressure. |
| **Active power-up + timer** | `state.powerUpTimers` | Near player or bottom-center | Shows which power-up is active and remaining duration. Disappears when no power-up is active. |

### Deliberately excluded from HUD

| Item | Reason |
|---|---|
| Phase name | Phases are felt through density/tempo, not read as text. A subtle progress bar is an acceptable alternative if phase-awareness is wanted — but not a label. |
| Coffee Break countdown | Diegetic — shown on Bus Stop signage in-world ("NEXT BREAK: 0:47"). No HUD duplication. |
| Monument count | Retrospective stat — belongs on end-of-run summary, not live HUD. |
| Full stat line | End-of-run summary only. |

### HUD reads GameState directly

UIScene reads `GameState` fields each frame — no event-driven HUD updates needed for the four elements above. This is consistent with the architecture spec's state-ownership table: "UI state (HUD values, menu state) → UIScene → Reads from GameState each frame, never writes to it."

## End-of-run summary

Displayed after SHIP IT door is touched (win) or quit-to-menu. Shows the full stat line from GDD §2.5:

| Stat | Source |
|---|---|
| Furthest-% reached | Derived: `(state.tick / TOTAL_TICKS) * 100`, capped at 100 |
| Deaths | `state.deaths` |
| Bits banked | `state.score` |
| Hazards outlived | Derived: count of hazard entities that despawned while player was alive during their lifetime |
| Net time edited | `state.timeTokenTally` |
| Monuments erected | `state.monumentsErected` (authoritative cumulative counter in GameState) |

"Monuments erected" is deliberately labeled a "shame stat" in the GDD — the summary presentation should lean into that tone.

### Share card

The summary data formatted for clipboard/share. Exact format is a design polish decision, not architecture. The data is all available from the final `GameState` snapshot.

## Death feedback

Split along the canonical/cosmetic boundary established in the architecture spec:

| Feedback | Mechanism | Layer |
|---|---|---|
| Death counter increment | UIScene reads `state.deaths` each frame — automatic, no event needed | Canonical (GameState read) |
| Screen shake / flash | Driven by `SimEvent { type: 'death' }` in `handleSimEvent()` | Cosmetic (event-driven, client-only) |
| Respawn animation | Driven by `SimEvent { type: 'respawn' }` | Cosmetic (event-driven, client-only) |

## Field Guide

An in-game screen accessible from Title screen and Pause overlay. A collection grid showing all entity types, with per-type near-miss survival tracked.

### Near-miss definition

A near-miss occurs when the player enters a hazard's expanded detection zone and exits alive without contacting the lethal AABB. Specifically:

- **Expanded zone:** hazard AABB expanded by `NEAR_MISS_MARGIN` in each dimension. Default: 1.5× the player's hitbox dimensions (33px horizontal, 45px vertical, given the 22×30 player size).
- **Trigger:** single-tick exit-alive — the player was inside the expanded zone on the previous tick and is outside it (or the hazard despawned) on the current tick, with no death event in between. No sustained-proximity requirement.
- **Emission:** `SimEvent { type: 'nearMiss', hazardType }` emitted at the exit tick. Client reacts by updating `SaveData.fieldGuide[hazardType] = true`.

`NEAR_MISS_MARGIN` is a named tuning constant, same status as `MAX_TICKS_PER_FRAME`, `RAMP_SECONDS`, or `COYOTE_TIME_TICKS` — expect it to move once someone plays the build. The margin reuses the same AABB-expansion pattern already established for placement spacing in the schedule-generation spec, not a novel proximity metric.

### Screen structure

Grid of entity type icons (procedural silhouettes from the Shape-Primitive Library). Discovered types show full color; undiscovered types show greyed-out silhouette. No detailed per-type stats — just "survived a near-miss: yes/no."

## Settings screen

Minimal for v1:

| Setting | Control | Persisted in |
|---|---|---|
| Master audio volume | Slider (0–100%) | `SaveData` (add `settings: { masterVolume: number }` — additive field, defaults to 100) |
| Controls reminder | Read-only display of keyboard/touch layout | Not persisted |

### Why audio volume is required

The game uses procedurally-generated Web Audio (ADR-0003) with no image assets (ADR-0002). Shipping synthesized audio with no volume control is a real usability gap — players need a way to adjust or mute without relying on OS-level controls.

## Daily Bin UI (prototype scope)

| Component | Prototype scope | Deferred |
|---|---|---|
| Mode-select (Classic / Daily Bin) | Built — pure UI, no backend dependency | — |
| Seed display | Built — shows date-derived seed string | — |
| Leaderboard viewing screen | — | Deferred until server exists. Do not build against mock data. |

## Acceptance criteria

- [ ] Screen flow matches: Title → Mode Select → Gameplay → End-of-Run Summary → Title, with Pause and Settings overlays
- [ ] HUD shows exactly: timer, bits banked, death count, active power-up + timer — nothing else during gameplay
- [ ] UIScene stays active during GameScene pause (Pause overlay renders correctly)
- [ ] End-of-run summary displays all six stats from GDD §2.5
- [ ] Death counter updates from `GameState.deaths` read, not from event
- [ ] Screen shake/flash on death is driven by `SimEvent { type: 'death' }`
- [ ] Field Guide screen is accessible from Title and Pause overlay
- [ ] Settings screen includes audio volume control that persists across sessions
- [ ] Daily Bin mode-select and seed display are functional without a backend
- [ ] No Coffee Break HUD element exists — countdown is diegetic (Bus Stop signage only)

## Dependencies

- [specs/features/architecture.md](architecture.md) — UIScene/GameScene split, SimEvent channel, GameState fields
- [specs/features/save-manager.md](save-manager.md) — SaveData shape, Field Guide persistence
