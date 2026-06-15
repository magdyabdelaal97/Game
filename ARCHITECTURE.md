# Genesis — Architecture

How the whole system fits together: the shared spine, the twelve layers, the exact
contract for each, and the build order. This is the technical bible.

## A. The spine (lives in `packages/kernel`)

Everything plugs into three kernel facilities. No layer may bypass them.

### A1. The state store (ECS)
The single authoritative copy of the world, organized as an **Entity-Component-System**:
- **Entity** = a bare id (a being, a place, an institution, an idea…).
- **Component** = a plain data record attached to an entity (e.g. `Position`,
  `Health`, `Beliefs`, `Membership`). Components are data only — no methods.
- **System** = a function (owned by a layer) that reads/writes components each tick.

Rules: layers read/write components **through the store API**, never via private
copies. Changing a shared component's shape requires updating this file in the same
commit. Each component is owned by exactly one layer (the only writer); others read.

### A2. The clock
Advances game time in **fixed, ordered ticks**. Supports a variable rate (see
CONCEPT §4) where the rate is part of the recorded action stream. Systems run in a
**declared, fixed order** each tick (the kernel owns the schedule). No system may
depend on real wall-clock time.

### A3. The event bus
Layers communicate by emitting and subscribing to **events** (plain serializable
records). A layer prefers emitting an event over calling another layer. Events are
processed deterministically (ordered queue drained per tick). Removing a subscriber
must never crash a publisher.

### A4. Seeded RNG
A seeded pseudo-random generator with named streams (e.g. `world`, `beings`,
`director`). **The only source of randomness in the entire simulation.** Never
`Math.random`. Same seed ⇒ same sequence.

### A5. Snapshots
Serialize the entire store + clock + RNG state to a snapshot, and restore it exactly.
This powers save/load, offline ticking, and the golden-seed replay harness.

---

## B. Contract format

Every layer below declares:
- **Consumes** — components it reads + events it listens for.
- **Produces** — components it writes (owns) + events it emits.
A layer touching anything outside its contract is a bug. The contract is the test
target (see TESTING.md).

---

## C. The twelve layers (build order = bottom to top)

Realms: **Foundation · World&Life · Society&Mind · Spirit&Story · Shell.**
Package names use `@genesis/<id-name>`.

### L0 · Kernel  — `@genesis/kernel`  (Foundation)
The spine above (A1–A5) plus a **debug inspector** API the client renders so the sim
is visible from day one.
- Consumes: `player.action`, `clock.config`.
- Produces: `clock.tick`; the `state.*` store; `snapshot`.

### L1 · World — `@genesis/world`  (World&Life)
Geography, biomes, climate, seasons, resources, ecology, natural events. Rule/math
based — a drought emerges from the climate model, not a story.
- Consumes: `clock.tick`.
- Produces: components `Terrain, Climate, ResourceNode, Ecology`; events
  `DroughtBegan, SeasonChanged, ResourceDepleted, DisasterStruck`.

### L2 · Beings — `@genesis/beings`  (World&Life)
Persistent identity for every resident: `age, lifeStage, Health, traits, Values,
Beliefs, Skills, Needs, Relationships, Memories, Ancestry, Reputation,
SpiritualDevelopment`. Owns birth/aging/death/inheritance and the **LOD tier**
field (A/B/C/D) + promotion/demotion (see SCALABILITY.md). Beings act on **bounded
knowledge** (what they know, possibly wrong).
- Consumes: `clock.tick`; `WorldEvent`s.
- Produces: components above; events `Born, Died, CameOfAge, NeedCritical`.

### L3 · Behaviour — `@genesis/behaviour`  (World&Life)
Turns identity into action. **Utility AI** scores wants; **GOAP** plans longer aims;
**HFSM/behaviour tree** runs immediate actions. Beliefs and mood feed the scores.
- Consumes: `Beings` components, `World` components, `clock.tick`.
- Produces: writes `BeingAction` (chosen act) back to state; events
  `ActedAgainstRuler, Migrated, Converted, CommittedCrime`.

### L4 · Bonds — `@genesis/bonds`  (Society&Mind)
Groups with their own interests: households, families, classes, factions, schools,
temples, guilds, councils. Membership, hierarchy, loyalty, group goals distinct from
members', institutional drift, corruption, reform.
- Consumes: `Beings` components, `BeingEvent`s.
- Produces: components `Institution, Membership, GroupGoal`; events
  `SuccessionCrisis, FactionFormed, InstitutionCorrupted, Reform`.

### L5 · Provision — `@genesis/provision`  (Society&Mind)
Economy + emergent knowledge. Production chains, labor, housing, storage, markets,
prices, trade, wealth distribution. Discovery emerges from
knowledge+materials+pressure+specialists+chance (no fixed tech tree).
- Consumes: `World` resources, `Beings` labor/skills, `Institution` control.
- Produces: components `Economy, Wealth, KnownTech`; events
  `Famine, TradeOpened, Discovery, PriceShock`.

### L6 · Culture — `@genesis/culture`  (Society&Mind)
Beliefs as living things. Each idea carries meaning, emotional strength, sacredness,
prestige, compatibility, supporting/opposing institutions. Transmission via parents,
teachers, charisma, conformity, trauma, prosperity, revelation, migration. Processes:
orthodoxy, heresy, syncretism, reformation, revival, secularization, mystical
reinterpretation. Encodes the competing goods (CONCEPT §5).
- Consumes: `Beings` beliefs, `Institution`, `EconEvent`, `SocialEvent`.
- Produces: components `Doctrine, BeliefLandscape`; events
  `Schism, Syncretism, Conversion, Secularisation, Revival`.

### L7 · Director — `@genesis/director`  (Spirit&Story)
The drama manager. Watches the sim (tension, conflicts, threats to the lineage,
cultural contradictions, lulls, overused story types) and fires **storylets**
(modular scenarios: preconditions, actors, stakes, escalations, resolutions,
consequences, exclusions). Seeded, variable pacing — structured unpredictability.
**Every storylet consequence maps to a real state change; it invents nothing.**
- Consumes: all `*Event` streams + `state.*` (to check preconditions).
- Produces: `Storylet.active`; `DirectorIntent` (where to focus player + Voice).

### L8 · Divine — `@genesis/divine`  (Spirit&Story)
The Deity's powers + the three-identity model + the Adamic lineage + the costed
intervention economy (attention/faith/grace, with over-use consequences). Divine
actions change state through the **same channels mortals use** — no privileged edits.
- Consumes: `Culture` (faith available), `Beings` (prophets/heirs), `DirectorIntent`.
- Produces: writes `DivineAction` as a real cause; events
  `OmenSent, ProphetRaised, DivineSilence, Reincarnation`; lineage components.

### L9 · Voice — `@genesis/voice`  (Spirit&Story)
LLM narration + the Chronicle. Receives **only validated facts** and renders speech,
scripture, dreams, letters, prophecy, and chronicle entries. Full memory/reflection
cognition only for **Tier-A** agents. **Read-only on state — may never invent a war,
death, marriage, or law.** Deleting this layer must leave the simulation unchanged.
- Consumes: validated fact-sets (read-only), `Storylet.active`.
- Produces: narration text (display only — never written back to authoritative state).

### L10 · Client — `@genesis/client`  (Shell)
What the player sees/touches: PixiJS 2D renderer, panels, the god's tools, the
Chronicle reader, AND the esoteric side-modes (Incarnation). Replaces the debug
inspector with a designed experience (inspector stays as a dev tool). Side-mode
results re-enter culture as artifacts. **UI reads state; it holds no authority.**
- Consumes: `state.*`, narration text, issues `DivineAction`.
- Produces: `player.action` (everything the player does → kernel).
- NOTE: the only package allowed to use DOM/PixiJS/`window`.

### L11 · Net — `@genesis/net`  (Shell)
Multiplayer + persistence. Authoritative server simulation, offline ticking (resume
from snapshots), accounts/social/matchmaking (Nakama), trade & diplomacy, shared
history/ruins. The server validates every client action; clients only propose.
- Consumes: `snapshot` (resume), `player.action` (validated server-side).
- Produces: the authoritative world; events `TradeOffered, AllianceFormed,
  CivilizationFell`.

---

## D. Data-flow rules (read twice)

1. Player input enters as `player.action` → kernel applies it → systems react on the
   next tick → events fan out → Director/Divine respond → Voice narrates the result →
   Client renders. The loop is one-directional into the authoritative state.
2. **Up only:** a layer may consume from layers below it and emit upward. It must not
   import or call a layer above it.
3. The Voice and Client sit *outside* the authoritative loop — they observe it.
4. Every consequence the player sees must correspond to a real state change recorded
   in the snapshot. If it isn't in state, it didn't happen.

---

## E. Recommended stack (swappable via contracts)

- **Sim + client + tests:** TypeScript (ES modules), pnpm monorepo, Vitest.
- **State store:** a small custom ECS (or an existing TS ECS) in the kernel.
- **Renderer:** PixiJS (2D). `InstancedMesh`/Three.js only if 3D is later chosen.
- **Sim runtime:** plain Node for tests; a Web Worker in browser (keeps UI smooth).
- **LLM:** an adapter in `voice` behind an interface (provider-swappable).
- **Memory (Tier-A):** Mem0 / Letta-style store behind a `voice` interface.
- **Backend (L11):** Nakama (accounts/social) + custom authoritative sim service.
- **Design lab (optional, throwaway):** Python + Mesa/Concordia to prototype society
  rules quickly before porting the proven version to the TS sim. Lab code never ships.

## F. Repository layout

```
packages/<layer>/   # one package per layer (L0..L11), public surface in index.ts
sim/                # composition root: wires layers onto the kernel bus
tools/replay/       # golden-seed replay harness
tools/lab/          # optional Python design sandbox (does not ship)
tests/golden/       # saved seed runs for regression
docs/  plans/  .claude/
```

## G. Phase order (one layer per phase; gate before next)

P0 Kernel → P1 World → P2 Beings → P3 Behaviour → P4 Bonds → P5 Provision →
P6 Culture → P7 Director → P8 Divine → P9 Voice → P10 Client → P11 Net.

A minimal **debug renderer ships in P0** so every later phase is visible. The polished
Client (P10) and Multiplayer (P11) come last on purpose — get the reproducible
single-player simulation correct first.
