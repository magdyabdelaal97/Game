# Genesis — Plain-Word Glossary

For terms used across the docs. Plain language for a non-game-dev maintainer.

- **Tick** — one fixed step of game time; the world advances in ordered ticks so
  results are repeatable.
- **Seed** — the starting number for the random generator; same seed → same "random"
  events, so any run can be reproduced.
- **Deterministic** — same inputs always give the same output; required for testing,
  replay, and fair multiplayer.
- **ECS (Entity-Component-System)** — store game data as plain components (position,
  health, beliefs) attached to lightweight entity ids; systems are functions that
  process them. Fast and easy to extend.
- **Event bus** — a shared message channel; layers announce things ("ChildBorn") and
  others listen, instead of calling each other directly.
- **Contract / interface** — the published list of what a layer reads (consumes) and
  writes (produces). Honor it and you can rewrite the layer's insides freely.
- **Component owner** — the single layer allowed to write a given component; others
  only read it.
- **Utility AI** — a decision method that scores each possible action by how well it
  meets current needs, then picks the best.
- **GOAP (Goal-Oriented Action Planning)** — the agent has a goal and a planner finds
  a sequence of actions to reach it, instead of hand-written scripts.
- **Behaviour tree / HFSM** — two ways to organize immediate actions and states
  (working, sleeping, fleeing); good for short, reliable behavior.
- **LOD (Level of Detail)** — spend more computation on what's near/important and less
  on what's distant; here a few beings are fully simulated, thousands are cheap data.
- **Promotion / demotion** — moving a being between LOD tiers, synthesizing a
  plausible history when a background being steps into the spotlight.
- **Storylet** — a small modular scenario with preconditions, actors, and outcomes;
  the Director fires these to build emergent stories.
- **Director / drama manager** — the system that watches the sim and paces the story
  by choosing which storylets/events occur and when.
- **RAG (Retrieval-Augmented Generation)** — giving the LLM relevant facts to ground
  its output instead of training it.
- **Authoritative state** — the one true copy of the world the kernel/server trusts;
  the LLM and UI may read it but never overrule it.
- **Golden-seed replay** — a saved run re-played after changes to check nothing broke;
  the same seed should yield the same world.
- **Hero / Tier-A agent** — an important being (ruler, prophet, rival) given full
  memory and LLM cognition, unlike the cheap majority.
- **Snapshot** — a serialized copy of the whole world that can be restored exactly.
- **Composition root** — the one place (`sim/`) that wires all layers together onto
  the kernel; keeps layers unaware of each other.
