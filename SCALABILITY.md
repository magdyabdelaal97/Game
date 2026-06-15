# Genesis — Scalability

How the game stays fast *and* maintainable as population, features, and players grow,
without rewrites. Every layer must respect these rules.

## 1. The core trick: Level of Detail (LOD) for agents

You cannot run rich per-being cognition (especially an LLM) on thousands of residents
in real time. Spend compute where the player looks. Every being carries a `tier`:

| Tier | Who | Count | Update | Cost |
|------|-----|-------|--------|------|
| **A — Hero** | Adamic family, religious/philosophical leaders, major merchants, rivals, current-storyline actors, player favorites | ~20–100 | Every tick; full memory, planning, LLM dialogue | High |
| **B — Active local** | Residents of the current settlement | hundreds–few thousand | Every few ticks; utility AI, simple plans | Medium |
| **C — Background** | Wider population | tens of thousands | Daily/weekly batched events | Low |
| **D — Distant cohorts** | Populations outside the active settlement | unbounded | Represented as aggregate numbers ("420 laborers, 36 dissident families") | Near-zero |

**Promotion / demotion** is mandatory and is owned by `beings` + `director`:
- When a being becomes relevant (player approaches, storyline selects them), they are
  **promoted** to a higher tier. If promoting from C/D, **synthesize a plausible
  history** consistent with their cohort, ancestry, and recorded events — so they feel
  like they always had an inner life.
- When relevance ends, **demote** them, persisting any important new memories as a
  compact summary.

The LLM (Tier-A only) is a **budgeted resource the Director allocates** — surface it
in the game as the Deity's "attention," so cost control and the god-fantasy reinforce
each other. Tier B/C/D cost ~zero LLM tokens.

## 2. Separate the simulation from the rendering (already enforced by Law III)

- Sim packages are pure and engine-agnostic; they run in a **Web Worker** in the
  browser so heavy ticking never freezes the UI, and in **plain Node** for tests.
- The client only reads state and draws it. You can upgrade graphics (debug shapes →
  PixiJS sprites → 3D) without touching the sim.
- For multiplayer, the same sim runs **server-authoritatively**; clients propose,
  the server decides. The sim code is identical — only where it runs changes.

## 3. Performance budgets (set targets per phase; measure, don't guess)

- Each tick has a **time budget** (e.g. target 60 ticks/sec headroom on a mid laptop
  with N active beings). Add a tick-timing readout to the debug inspector in P0.
- If a system blows its budget, fix with (in order): (a) batch the work, (b) run it
  less often (lower tier / lower cadence), (c) spatial partitioning, (d) cache.
  **Never** fix it by breaking determinism (e.g. "skip some randomly").
- Memory: cap each being's memory stream; summarize and forget on demote; never let a
  stream grow unbounded (the known failure mode of naïve generative agents).

## 4. Scaling without rewrites (why the architecture allows it)

- **One layer = one package with a contract.** To make a layer faster or smarter, you
  rewrite *inside* that package. As long as its consumes/produces contract is
  unchanged, nothing else needs to change, and the golden-seed replay proves you
  didn't alter behavior unintentionally.
- **Events, not direct calls.** Adding a new layer that reacts to existing events
  requires zero changes to existing layers.
- **Components are owned by one writer.** Adding a new component never breaks existing
  systems; they simply don't read it.
- **The sim is headless.** Swapping PixiJS for another renderer, or adding a server,
  touches only `client`/`net`.

## 5. Determinism is a scalability feature, not just a testing one

Because the world is reproducible from `seed + version + actions`:
- You only ever store the **seed + action log**, not the whole world, to reproduce any
  run — cheap to log, cheap to debug, cheap to share bug reports.
- The server can **fast-forward offline time** deterministically and resume from
  snapshots.
- Multiplayer trust is simpler: both sides agree on the rules, the server replays the
  authoritative stream.

## 6. Anti-patterns the agent must refuse

- Running an LLM call inside a per-being loop over the whole population. (Tier-A only.)
- A layer caching another layer's state and letting it drift out of sync.
- Using `Math.random` / `Date.now` / unordered iteration anywhere in a sim package.
- "Solving" a perf or test problem by introducing nondeterminism.
- Putting authoritative logic in the client (it must be re-validatable on the server).
- Unbounded growth: memory streams, event logs in RAM, entity counts with no LOD.

If a requested feature would force one of these, stop and surface the tradeoff.
