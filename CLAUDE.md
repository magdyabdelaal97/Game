# Genesis — Agent Operating Manual

Genesis is a god-game and generational civilization simulator. It is built **one
layer at a time**: each layer is an isolated, separately-testable package that
talks to others only through contracts and an event bus. The goal is a codebase
the (non-expert) owner can maintain and extend for years — never a tangle.

**Read these before doing anything non-trivial** (load on demand, don't keep all in context):
- Game design → `@docs/CONCEPT.md`
- Layers, contracts, event bus, state → `@docs/ARCHITECTURE.md`
- How we scale without rewrites → `@docs/SCALABILITY.md`
- How we test + the gate ritual → `@docs/TESTING.md`
- Current phase plan → the active file in `@plans/` (e.g. `plans/PHASE-00-kernel.md`)

---

## The Five Laws (non-negotiable; if a change breaks one of these, STOP and ask)

1. **One source of truth.** The simulation state (in `packages/kernel`) is the
   only authority. The LLM (`packages/voice`) and the UI (`packages/client`) may
   **read** state but MUST NOT write or invent it. The Voice narrates facts it is
   given; it never decides what is true.
2. **Determinism by seed.** Same seed + same code version + same actions ⇒ the
   identical world. In simulation code: **never** use `Math.random`, `Date.now`,
   wall-clock time, or unordered iteration. Use the kernel's seeded RNG and clock.
3. **Plug by contract.** A layer communicates only through its published interface
   and the event bus. It MUST NOT import another layer's internals or hold private
   copies of shared state. Honor the contract and any layer can be rewritten freely.
4. **Level of detail.** Most beings are cheap data; only a handful run full
   (Tier-A) cognition at once. Never run expensive per-being work on the whole
   population. See `@docs/SCALABILITY.md`.
5. **Pass the gate, then build.** A layer is done when its tests pass AND its
   contract holds — not when it "looks done." Show test output as evidence. Do not
   start the next phase until the current gate is green.

---

## Workflow for every phase (follow in order)

1. **Explore (plan mode).** Read the active `plans/PHASE-NN-*.md`, the layer's
   section in `docs/ARCHITECTURE.md`, and any layers it depends on. Use a subagent
   to explore so the main context stays clean.
2. **Plan (plan mode).** Confirm the file list, the contract (consumes/produces),
   and the exact tests. If the phase plan is missing detail, fill it using
   `plans/PHASE-TEMPLATE.md` and ask before coding.
3. **Implement.** Build the smallest slice that satisfies the contract. Write tests
   alongside, not after.
4. **Verify (YOU MUST run, never claim).** Run the commands below and paste the
   real output. "Tests pass" without output is not acceptable.
5. **Review.** Use the `contract-reviewer` subagent to check the diff against the
   phase plan and the layer contract. Fix gaps that affect correctness.
6. **Gate + commit.** Confirm every gate item in the phase plan, then commit with a
   message naming the phase and layer.

IMPORTANT: one phase per session. Run `/clear` between phases.

---

## Commands (this repo)

- Install: `pnpm install`
- Run all tests: `pnpm test`
- Test one package: `pnpm --filter @genesis/<layer> test`
- Determinism / golden-seed replay: `pnpm replay`
- Type-check: `pnpm typecheck`
- Lint: `pnpm lint`
- Run the dev client (debug inspector): `pnpm dev`

After any series of changes: run `pnpm typecheck` and the relevant `pnpm --filter … test`.
Prefer running a single package's tests over the whole suite during iteration.

---

## Conventions

- **Language:** TypeScript, ES modules. One language across sim, client, and tests.
- **Monorepo:** pnpm workspaces. One package per layer under `packages/`. A layer's
  public surface lives in its `index.ts`; everything else is private to it.
- **Sim is engine-agnostic & pure:** `packages/kernel` and all sim layers run in
  plain Node (for tests) and in a Web Worker (in the browser). No DOM, no PixiJS,
  no `window` in any sim package. Rendering lives only in `packages/client`.
- **Events over calls:** layers emit/subscribe on the kernel event bus. Prefer
  adding an event over importing another layer.
- **State shape changes** to shared components require updating
  `docs/ARCHITECTURE.md` in the same commit.
- Prefer pure functions and explicit inputs over hidden global state.
- Prefer small, named modules over large files.

## Hard "prefer X over Y" rules (these get ignored if phrased as "don't")

- Prefer the kernel RNG over `Math.random` — always, in any sim package.
- Prefer emitting an event over reaching into another package.
- Prefer extending a contract (and its tests) over a quick cross-layer hack.
- Prefer writing a failing test that reproduces a bug before fixing it.

If you catch yourself wanting to break a Law to "make it work," that is the signal
to stop and surface the tradeoff to the owner instead.
