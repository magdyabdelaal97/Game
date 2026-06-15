# Phase 00 — Kernel (`@genesis/kernel`)

> This phase is fully written as the worked example. Later phases follow
> `plans/PHASE-TEMPLATE.md` at this level of detail. Owner approves before coding.

## Objective
Build the deterministic spine every other layer plugs into: a fixed-step clock, a
seeded RNG, an ECS state store, an event bus, snapshot save/load, and a tiny **debug
inspector** so the simulation is visible from the very first phase. No game content
yet — just the engine that makes content reproducible and testable.

## Depends on
Nothing. This is the foundation. Everything else depends on it.

## In scope
- Fixed-step `Clock` with a settable rate; rate changes recorded in the action log.
- `RNG` with named, seeded streams (`world`, `beings`, `director`, …); the **only**
  randomness in the project.
- `Store` (ECS): create/destroy entities; add/get/remove components; query by
  component; a single component-owner registry.
- `EventBus`: emit, subscribe, deterministic ordered drain per tick.
- `snapshot()` / `restore()`: serialize and restore Store + Clock + RNG exactly.
- `applyAction(action)`: the single entry point for `player.action`, appended to a
  recorded action log.
- `DebugInspector`: a read-only API exposing tick count, entity/component counts, and
  a tick-time reading, plus a tiny dev page (`pnpm dev`) that prints them live.
- Project scaffolding: pnpm workspace, Vitest, ESLint/Prettier, `tsconfig`, CI, and
  the `pnpm replay` harness skeleton in `tools/replay`.

## Out of scope (explicitly)
- No world, beings, behavior, or any game systems.
- No PixiJS rendering (the inspector is plain text/numbers for now).
- No networking, no LLM, no save UI.

## Contract (matches docs/ARCHITECTURE.md · L0)
- **Consumes:** `player.action`, `clock.config`.
- **Produces:** `clock.tick` (event each step); the `state.*` Store; `snapshot`.
- New shared types introduced here: `Action`, `Event`, `Snapshot`, `ComponentId`,
  `EntityId`. Document them in ARCHITECTURE.md §A.

## File layout
```
packages/kernel/
  src/index.ts          # public surface: Clock, RNG, Store, EventBus, Kernel, DebugInspector
  src/clock.ts          # fixed-step clock + rate
  src/rng.ts            # seeded PRNG (e.g. mulberry32/xoshiro) + named streams
  src/store.ts          # ECS: entities, components, queries, owner registry
  src/eventBus.ts       # emit/subscribe/ordered drain
  src/snapshot.ts       # serialize/restore store+clock+rng
  src/kernel.ts         # wires the above; applyAction + tick loop
  src/inspector.ts      # read-only debug readout
  test/*.test.ts
sim/index.ts            # composition root (empty wiring for now)
tools/replay/index.ts   # golden-seed replay harness
tests/golden/           # (created; first golden run added here)
.github/workflows/ci.yml
```

## Build steps (smallest slice first)
1. Scaffold the pnpm monorepo, Vitest, lint, tsconfig, CI.
2. `RNG`: implement a seeded PRNG with named streams; test determinism.
3. `Store`: entities + components + queries + owner registry; test add/get/remove.
4. `EventBus`: emit/subscribe + deterministic ordered drain; test ordering.
5. `Clock`: fixed-step advance + settable rate; emits `clock.tick`.
6. `Kernel`: tie Clock+RNG+Store+EventBus together; `applyAction` appends to the
   action log; a `tick()` drains events deterministically.
7. `snapshot()` / `restore()`: round-trip the whole state; test equality.
8. `DebugInspector` + `pnpm dev` page printing tick count, counts, tick-time.
9. `tools/replay`: run `{seed, version, actions}` → snapshot hash; compare to expected.
10. Add the first golden run (a few no-op/action ticks) to `tests/golden/`.

## Tests (all four kinds)
- **Unit:** RNG sequence is repeatable; Store add/get/remove correct; EventBus drains
  in order; Clock advances by rate; snapshot→restore yields an equal Store.
- **Contract:** `clock.tick` fires once per step; `applyAction` records every action;
  attempting to write a component without owning it throws.
- **Determinism:** run 1,000 ticks twice with the same seed + actions → identical
  snapshot hash. Add this as `tests/golden/0001-kernel-baseline`.
- **Visual:** `pnpm dev` shows the tick counter climbing and counts updating; "save →
  reload → replay" returns the identical state.

## Verification commands (run and paste output)
```
pnpm install
pnpm typecheck
pnpm lint
pnpm --filter @genesis/kernel test
pnpm replay
```

## Gate / Definition of Done
1. typecheck + lint clean.
2. All kernel unit/contract/determinism tests pass (output shown).
3. `pnpm replay` matches on the new golden run (output shown).
4. Same-seed 1,000-tick run proven byte-identical across two runs.
5. snapshot→restore round-trips exactly.
6. `pnpm dev` inspector visibly shows live tick + counts + tick-time.
7. CI is green on a push.
8. contract-reviewer subagent reports no correctness/contract gaps.
9. ARCHITECTURE.md §A updated with the new shared types.

## What you'll see when it works
A bare dev page: a tick number climbing, "entities: 0, components: 0", a tick-time in
milliseconds, and a button sequence that saves the world, reloads it, and replays it —
returning the *exact* same numbers. Ugly on purpose. This is the heartbeat everything
else plugs into.

## Open questions for the owner
- Confirm TypeScript + pnpm + Vitest + PixiJS-later as the stack (vs. an engine).
- Confirm target tick budget for the inspector readout (e.g. 60/sec headroom).
- Pick the PRNG algorithm preference (default: mulberry32 for simplicity).
