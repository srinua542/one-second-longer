# Ephemeral event channel between simulation and presentation

`simulation.step()` emits gameplay events (entity spawned, death, power-up collected, etc.) by pushing to a caller-owned `SimEvent[]` array passed as an optional parameter. The client reads these events once per frame for view lifecycle management and cosmetic triggers (VFX, SFX, camera shake), then discards them. The server omits the collector entirely — zero allocation overhead during replay verification.

Events are typed as a discriminated union so the compiler enforces exhaustive handling when new event types are added.

This is the standard simulation/presentation event-channel split used in Overwatch (Tim Ford, GDC 2017), Unity Animation Events, and Unreal's Gameplay Ability System notifies — not event-sourcing/CQRS, which is a storage pattern. The distinction matters because events here are ephemeral derived output: never stored in `GameState`, never included in the replay hash, never fed back as input to a future `step()` call.

The alternative — pure state diffing — was considered and retained as a secondary bootstrap mechanism (scene creation/recreation) but rejected as the primary per-frame driver. Diffing can tell you *what* changed but not *why*: it cannot distinguish a death by hazard from a timer despawn from a broom-sweep, and the GDD requires ~15 distinct cosmetic reactions that depend on causality. Inferring causality from deltas is fragile reverse-engineering that the simulation already knows at the mutation site.

If a future feature needs "did event X happen recently," that is a signal to add a counter or flag to `GameState` (per ADR-0008's "store facts, derive conclusions" rule) — not a reason to persist the event log.
