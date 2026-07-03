# Canonical state vs generation output vs derived view

All simulation data is split into three layers with distinct lifecycles: **Generation Output** (pre-rolled schedule, computed once from the seed, immutable, never hashed), **GameState** (canonical mutable state, stepped every tick, this is what replay verification hashes), and **Derived View** (pure functions of GameState, recomputed on demand, never stored or hashed).

The alternative — a single `GameState` struct containing everything including the pre-rolled schedule and convenience booleans — was rejected for two reasons. First, the schedule is a pure function of the seed; storing it in the hashed state makes every schedule-format change a snapshot-breaking change, even though the simulation hasn't diverged. Second, storing derived flags (e.g. `mercyRuleActive`) alongside their source data (`recentDeathCounter`) creates two sources of truth that can silently drift apart between client and server — exactly the desync risk that deterministic replay verification exists to prevent.

This is the same discipline used in Factorio's desync-detection checksums, GGPO rollback save-state contracts, and Age of Empires lockstep netcode: hash only what's irreproducible without replaying every tick. The governing principle is "store facts, derive conclusions."

Consequences: every field added to `GameState` must justify why it can't be regenerated from the seed or computed from existing fields. Derived values live as co-located pure functions (`isMercyRuleActive(state)`), never as stored booleans. Unbounded logs in `GameState` (death log, avoidance log) must be capped to what consuming mechanics actually read.
