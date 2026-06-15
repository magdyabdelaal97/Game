# Genesis — Testing Doctrine & The Gate

Testing is how we avoid an unmaintainable game. **A layer is not "done" because it
runs; it is done when its tests pass and its contract holds.** This file defines the
four kinds of test every layer needs, the golden-seed replay, and the gate ritual.

## 0. The evidence mandate (most important rule)

The agent MUST run checks and show their output. A claim like "tests pass" without
the command and its result is not acceptable and must be rejected. Every phase ends
by pasting:
- the exact command(s) run, and
- their real output (pass/fail counts, replay diff result).

This single discipline is what turns an unattended run into a trustworthy one.

## 1. The four kinds of test (every layer has all four)

| Kind | Question | What it does |
|------|----------|--------------|
| **Unit** | Does its own logic work? | Tests one layer's rules in isolation (a hungry being eats; scarcity raises prices; an institution can corrupt). Fast, many, run constantly. |
| **Contract** | Does it honor its interface? | Asserts the layer consumes/produces exactly the events and components in its `docs/ARCHITECTURE.md` contract — and that it never writes components it doesn't own, nor imports another layer's internals. This is what guarantees pluggability. |
| **Determinism** | Is it reproducible? | Runs the layer twice with the same seed + inputs and asserts identical output. |
| **Visual / manual** | Can you *see* it work? | A human watches the debug inspector / a small view and confirms the behavior reads correctly. Catches "passes but feels wrong." |

Use **Vitest**. Each package has its own `test` script. Prefer running a single
package's tests during iteration; run the full suite before a gate.

## 2. The golden-seed replay (regression safety net)

Because the engine is deterministic (Law II), we keep a small library of saved runs in
`tests/golden/` — each is a `{ seed, version, actions[] }` plus an expected snapshot
hash. `pnpm replay`:
1. loads each golden run,
2. re-executes the action stream from the seed,
3. compares the resulting snapshot hash to the expected one.

If it matches → behavior unchanged (safe). If it differs when it shouldn't → you
caught a regression instantly. When a change *intentionally* alters behavior, you
regenerate and review the new golden hashes as part of the commit. **Add at least one
golden run per phase.**

A divergent `pnpm replay` is the highest-priority bug class — it means determinism
(Law II) broke, and everything else depends on it.

## 3. Per-layer test requirements (minimum bar)

Each phase plan lists its specific tests; these are the floors:
- **Unit:** the layer's central rule works and an obvious failure case is handled.
- **Contract:** at least one test per produced event + per owned component; one test
  asserting it ignores/does-not-write out-of-contract state.
- **Determinism:** one same-seed-twice test in the package; one new golden run added.
- **Visual:** a note in the phase plan describing what the owner should see in the
  inspector, plus (where practical) a snapshot/fixture test of the rendered debug data.

## 4. The gate ritual (run before declaring a phase done)

Use the `/phase-gate` skill, which walks this checklist and refuses to pass until all
are green:
1. `pnpm typecheck` — clean.
2. `pnpm lint` — clean.
3. `pnpm --filter @genesis/<layer> test` — all pass (output shown).
4. `pnpm replay` — all golden runs match (output shown).
5. Contract tests present for every produced event/component.
6. A new golden run for this phase exists and is committed.
7. The `contract-reviewer` subagent has reviewed the diff against the phase plan and
   reported no correctness/contract gaps.
8. The phase plan's "what you'll see" has been manually confirmed in the inspector.
9. `docs/ARCHITECTURE.md` updated if any shared component shape changed.

Only when all nine pass does the next phase begin.

## 5. Independent review (avoid the agent grading its own homework)

Before the gate, run a fresh-context review:
> "Use the contract-reviewer subagent to review the diff against
> plans/PHASE-NN-<layer>.md and the layer's contract in docs/ARCHITECTURE.md.
> Check every requirement is implemented, every produced event/component has a test,
> and nothing outside scope changed. Report only gaps that affect correctness or the
> contract — not style."

A reviewer asked for gaps will always find some; fix the ones that affect correctness
or the contract, treat the rest as optional (avoid over-engineering).

## 6. CI (set up in P0, extend each phase)

A CI workflow runs on every push: `pnpm install && pnpm typecheck && pnpm lint &&
pnpm test && pnpm replay`. A red build blocks merge. Optionally enforce locally with a
Stop hook (see `.claude/settings.example.json`) so a turn can't end while tests fail.

## 7. The simulation lab (optional, for design — not shipped)

Society/economy/belief *rules* can be prototyped fast in `tools/lab` (Python + Mesa /
Concordia) to validate behavior against historical ranges before porting the proven
rule into the TS sim. Lab experiments are throwaway and are not part of the gate.
