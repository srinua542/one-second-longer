# Save Manager

**Path:** features/save-manager
**Scope:** Client-side persistent data — schema shape, versioning policy, write timing, UUID identity. Excludes portal-specific cloud sync (deferred), server-side replay verification, and leaderboard submission.

## Save data shape

```typescript
interface SaveData {
  playerUUID: string;
  saveVersion: number;

  classic: {
    bestFurthestPct: number;
    fewestDeathsOnBestRun: number;   // tied to the best-furthest run, not lifetime lowest
    totalRuns: number;
    totalDeaths: number;
    totalBitsBanked: number;
  };

  dailyBin: {
    lastPlayedDate: string;          // ISO date string — detect "new day, new seed" on load
    todayBestScore: number | null;
    lifetimeRunsPlayed: number;
    currentStreak: number;           // cached copy — server-authoritative, see below
  };

  fieldGuide: Record<string, boolean>;   // key = EntityType, value = near-miss survived
  cosmetics: {
    devCoins: number;
    unlocked: string[];              // cosmetic IDs
    equipped: string | null;         // single slot unless GDD specifies multiple slots
  };
}
```

### Classic vs. Daily Bin split

Classic's "best" stats are lifetime persistent — the seed never changes, so improvement is cumulative. Daily Bin's `todayBestScore` resets each day (detected by comparing `lastPlayedDate` against the current date on load). Lifetime aggregates (`lifetimeRunsPlayed`) persist across days.

### Open questions (GDD dependencies)

- **Multiple cosmetic slots** — if the GDD later specifies separate player-skin and UI-theme cosmetics, `equipped` becomes `Record<string, string | null>` instead of a single string. Current shape assumes one slot.

### `currentStreak` — server-authoritative, client-cached

**Decision: included, minimal scope.** A bare counter displayed on the title screen / Daily Bin mode-select. No rewards or unlocks gated by streak length. No punishment UI for breaking a streak.

**Source of truth is the server**, not `SaveManager`:

- `currentStreak` is computed server-side, keyed by `playerId`, updated when a Daily Bin submission passes replay verification and gets written to Postgres.
- Date comparison uses the server's daily-reset boundary (same UTC date logic that rolls `leaderboard:{date}` in Redis) — not client local time — so streak and leaderboard day always agree.
- Streak increments only on a **verified valid submission** for that day, not merely "app opened" or "run attempted."

`SaveManager.dailyBin.currentStreak` is a **cached copy** for instant display, refreshed from the server response after each successful submission. Same sync pattern as best-stats — no new architecture needed.

#### Transactional write (named rule)

The streak update must happen in the **same Postgres transaction / Redis pipeline** as the leaderboard write (score → Postgres row + Redis sorted-set ZADD + streak increment). Not a separate follow-up call. If the streak write were a second round-trip that could fail independently, a verified score would land on the leaderboard with a stale streak — a small inconsistency with no re-verification path for streak alone. Single-transaction-or-nothing eliminates this class of partial-write bug.

**Why server-authoritative:** A client-only streak computed from `lastPlayedDate` can be inflated by changing the device clock — the same category of client-trusted claim that ADR-0006 (replay verification) was built to prevent. Inconsistent to verify scores server-side but trust streak values from the client.
- **Multiple cosmetic slots** — if the GDD later specifies separate player-skin and UI-theme cosmetics, `equipped` becomes `Record<string, string | null>` instead of a single string. Current shape assumes one slot.

## Schema versioning

**Default policy: additive-only.** New fields default to sensible zero/null values and get backfilled on the next write. Never rename or remove fields without a migration. Never wipe-on-mismatch.

**`saveVersion` stamp** is reserved for the rare case a field's *meaning* changes (not just a new field appearing). When a structural break is unavoidable:

1. Bump `saveVersion`
2. Write an explicit migration function keyed by version: `migrate_1_to_2(data): SaveData`
3. If the migration path itself is missing (e.g., `saveVersion` is 1 but no `migrate_1_to_2` exists in the deployed code), **log the error and preserve the raw data** rather than silently discarding it

### Why not wipe-on-mismatch

A botched save migration destroys Field Guide progress and cosmetic unlocks — irreversible harm to the player, worse than the `simVersion` mismatch case (which only rejects a single leaderboard score). The cost of carrying migration functions is low; the cost of data loss is high.

## Write timing

### Trigger: discrete bank events, debounced

A tab-close mid-run should not erase Piggy Bank deposits — that's the implied promise of banking. Writes are triggered from the **client layer** (not inside `simulation/step()`) in reaction to bank-related `SimEvent` emissions, debounced to a maximum of one write per second even if multiple bank events land close together.

```
SimEvent 'bitsBanked' → client handleSimEvent() → SaveManager.scheduleSave()
                                                    ↓
                                         debounce gate (≤1 write/sec)
                                                    ↓
                                         localStorage.setItem(KEY, JSON.stringify(data))
```

**Why client-side, not in `step()`:** `SimEvent` is ephemeral — consumed by the client and discarded (ADR-0009). Persistence logic reacting to events belongs in the client layer alongside other event consumers (view sync, cosmetic triggers), not in the deterministic simulation core. Synchronous `localStorage` writes inside `step()` would also introduce blocking I/O into the hot loop.

SaveManager's write trigger lives entirely in `client`, reacting to consumed `SimEvent`s. `simulation/` has no knowledge that saves exist — it never calls, references, or is aware of `SaveManager`. This is the same discipline as ADR-0009's "events are ephemeral, never fed back as input" rule, applied to a persistence consumer: one enforced principle (events are write-only outputs, never create coupling back into the simulation) with two applications (view sync and save persistence).

### Unconditional run-end write

On session completion (SHIP IT door touched, or quit-to-menu), `SaveManager` writes unconditionally as a final consistency pass — catches any stat updates from the final ticks that fell within the debounce window.

## Player UUID

Generated once via `crypto.randomUUID()` on first `SaveData` creation. Stored in the save blob, persists across sessions.

### Determinism safety

`crypto.randomUUID()` is safe here — this UUID is client identity metadata, never enters `GameState`, never affects a replay hash. The determinism prohibition from the architecture spec ("no `crypto.randomUUID()` in `simulation/`") does not apply to `SaveManager`, which lives entirely in `client`.

### localStorage fragility (accepted limitation)

Private browsing or cache wipes create orphaned identities — the same human gets a new UUID. This is explicitly accepted:

- The tech-stack doc's own phrasing — "pseudonymous UUID in localStorage **or portal SDK user ID if available**" — names the mitigation: prefer portal SDK identity when available (survives cache clears, tied to platform account). Fall back to localStorage UUID only in `NullAdapter`/self-hosted case.
- The fallback's fragility is a documented trade-off, not something to solve via fingerprinting (privacy-invasive for marginal gain in a casual leaderboard game).

## Storage key and format

Single key in `localStorage`. JSON blob. No compression needed — the full save structure is trivially small (well under 1KB even with a complete Field Guide).

```typescript
const SAVE_KEY = 'one-second-longer-save';

function load(): SaveData | null {
  const raw = localStorage.getItem(SAVE_KEY);
  if (!raw) return null;
  const data = JSON.parse(raw);
  return migrate(data);  // apply any pending version migrations
}

function save(data: SaveData): void {
  localStorage.setItem(SAVE_KEY, JSON.stringify(data));
}
```

## Acceptance criteria

- [ ] `SaveData` is a single JSON blob in `localStorage` under a fixed key
- [ ] `saveVersion` stamp is present; new fields default to zero/null without requiring a migration
- [ ] No save data is ever silently discarded on version mismatch — unknown versions are logged and raw data preserved
- [ ] `playerUUID` is generated once via `crypto.randomUUID()` on first save creation, never regenerated
- [ ] No `SaveManager` code exists inside `simulation/` — all persistence is client-side, triggered by `SimEvent` reactions
- [ ] Mid-run Piggy Bank deposits persist across a tab-close — verified by banking bits, closing the tab, reopening, and checking the save blob
- [ ] Write frequency is debounced to ≤1 write/second during gameplay
- [ ] Run-end triggers an unconditional save write regardless of debounce state
- [ ] Classic stats are lifetime-persistent; Daily Bin `todayBestScore` resets on date change

## Dependencies

- [specs/features/architecture.md](architecture.md) — SimEvent channel, client state ownership, GameState fields
- [specs/features/schedule-generation.md](schedule-generation.md) — EntityType union (keys for Field Guide)
