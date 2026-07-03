# Monorepo with shared simulation core

The codebase is a monorepo with three pnpm workspace packages: `packages/simulation/` (pure TypeScript, zero Phaser/DOM/Node dependencies), `packages/client/` (Phaser 4 + Vite), and `packages/server/` (Fastify + PostgreSQL + Redis). Both client and server import simulation as a workspace dependency.

This structure exists to make one guarantee physically enforceable: the client and server run *literally the same* simulation code — not a re-implementation, not a copy, the same imported module. This is non-negotiable because replay verification (ADR-0005) computes authoritative leaderboard scores by re-simulating a player's input log server-side. If the client and server simulations ever diverge — even by a single floating-point rounding difference in one collision check — legitimate players get rejected scores and the Daily Bin leaderboard loses integrity.

Considered alternatives: a single-package repo with the simulation as an internal directory (simpler, but no structural enforcement against accidental Phaser imports in simulation code); separate repos with a published shared package (introduces version drift risk — exactly the failure mode this decision prevents).

The monorepo adds workspace configuration overhead. This is accepted as cheap insurance against a failure mode that would be silent, data-corrupting, and extremely difficult to diagnose after real leaderboard data exists.
