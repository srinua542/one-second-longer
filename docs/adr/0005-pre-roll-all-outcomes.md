# Pre-roll all outcomes at schedule generation time

The spawn director generates a fully-resolved timeline before the session starts. Every spawn event's type, position, parameters, and chaos outcome (Mystery Crate contents, Gacha Door destinations, Weather Vane effects, Dice faces) are determined at generation time using seeded PRNG sub-streams. No PRNG calls occur at runtime except the isolated cosmetic stream (visual jitter, audio pitch variation).

The alternative — rolling chaos outcomes on player interaction — was rejected because it breaks Daily Bin fairness. If a Mystery Crate's content is determined by when the player opens it relative to other PRNG draws, two players with the same seed get different crate contents depending on interaction order. This violates the design rule "the stream is fixed; only the player varies" (GDD §2.6) and makes the leaderboard's same-seed comparison meaningless.

Player-history-dependent objects (the Regifter, the Understudy, the Complaint Form) are not exceptions — they use zero runtime randomness. They are deterministic lookups against the player's actual run history (avoidance log, death log). Tie-breaks, where needed, are pre-rolled as reserved draws in the chaos sub-stream during generation.

Consequence: the entire session is a fully-resolved data structure at boot. The solvability veto (GDD §6.2) can validate it headlessly. Replay verification (ADR-0005) can re-simulate it server-side. The runtime engine is a pure playback machine reacting to player input against a fixed schedule.
