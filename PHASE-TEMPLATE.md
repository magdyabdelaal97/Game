# Phase NN — <Layer name> (`@genesis/<layer>`)

> Copy this template to `plans/PHASE-NN-<layer>.md` and fill every section. Keep it to
> roughly one page — it is the owner's steering wheel and must be readable. Approve it
> before any code is written.

## Objective
One or two sentences: what this layer must do, in plain terms.

## Depends on
Which layers must already have passed their gate (and why this can't start sooner).

## In scope
The smallest set of capabilities that satisfies the contract for this phase.

## Out of scope (explicitly)
What this phase will NOT do, so the agent doesn't gold-plate or drift upward.

## Contract (must match docs/ARCHITECTURE.md)
- **Consumes:** components read + events listened for.
- **Produces:** components owned/written + events emitted.
- Any new shared component shapes (update docs/ARCHITECTURE.md in the same commit).

## File layout
```
packages/<layer>/
  src/index.ts          # public surface only
  src/<...>.ts          # internals (private to this package)
  test/<...>.test.ts
```
List the files to create and one line on each.

## Build steps
Numbered, smallest-slice-first. Each step should be independently runnable/testable.

## Tests (all four kinds — see docs/TESTING.md)
- **Unit:** …
- **Contract:** one per produced event/component + one "ignores out-of-contract" test.
- **Determinism:** same-seed-twice test + a new golden run in `tests/golden/`.
- **Visual:** what the owner should see in the debug inspector.

## Verification commands (the agent MUST run these and paste output)
```
pnpm typecheck
pnpm --filter @genesis/<layer> test
pnpm replay
```

## Gate / Definition of Done
A checklist; every item must be green (mirrors docs/TESTING.md §4). The phase is not
done until all pass and the contract-reviewer subagent reports no correctness/contract
gaps.

## What you'll see when it works
A concrete, plain-language description of the observable result in the inspector.

## Open questions for the owner
Anything to confirm via AskUserQuestion before coding.
