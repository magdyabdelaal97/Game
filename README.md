# Genesis — Owner's Guide (how to build this with Claude Code)

This repository is a **build plan, not yet a game**. It is structured so a Claude
Code agent builds the game layer by layer, testing each before the next, while you
stay in control even with only basic programming skills. This guide explains how to
drive it.

## What's here

```
genesis/
├── CLAUDE.md              # the agent reads this every session — the rules
├── README.md             # you are here
├── docs/                 # the "bibles" the agent reads on demand
│   ├── CONCEPT.md        #   the full game design
│   ├── ARCHITECTURE.md   #   the 12 layers, their contracts, the event bus
│   ├── SCALABILITY.md    #   how it stays fast and maintainable as it grows
│   ├── TESTING.md        #   how every layer is tested + the gate ritual
│   └── GLOSSARY.md       #   plain-word definitions of all jargon
├── plans/
│   ├── PHASE-TEMPLATE.md  # the shape every phase plan must take
│   └── PHASE-00-kernel.md # the first phase, fully written, as the worked example
└── .claude/
    ├── agents/           # specialist reviewers the agent delegates to
    │   ├── explorer.md
    │   ├── contract-reviewer.md
    │   └── test-runner.md
    ├── skills/phase-gate/SKILL.md  # the gate checklist workflow (/phase-gate)
    └── settings.example.json       # hooks that enforce testing automatically
```

## First-time setup (once)

1. Install Claude Code and `pnpm` and Node 20+.
2. Open this folder in Claude Code. It auto-loads `CLAUDE.md`.
3. Copy `.claude/settings.example.json` to `.claude/settings.json` to turn on the
   hooks (auto-typecheck/test after edits). Optional but recommended.
4. Run `/init` if you want Claude to refine `CLAUDE.md` to your machine — but the
   provided one is already tuned; keep it short.

## The loop you repeat for each phase

The phases are listed in `docs/ARCHITECTURE.md` (P0…P11, one per layer). For each:

1. **Start a fresh session.** Run `/clear` so context is clean. Name it, e.g.
   `/rename phase-00-kernel`.
2. **Generate the phase plan (if it doesn't exist yet).** Phase 0 is already
   written. For later phases, ask in **plan mode**:
   > "Read docs/ARCHITECTURE.md for the <Layer> layer and plans/PHASE-TEMPLATE.md.
   > Write plans/PHASE-0N-<layer>.md following the template. Interview me on any
   > open decisions using AskUserQuestion before writing it."
   Review the plan it writes (press Ctrl+G to edit it yourself). **This is where you
   stay in control** — the plan is short and readable.
3. **Implement.** Switch out of plan mode:
   > "Implement plans/PHASE-0N-<layer>.md. Write tests alongside. Run
   > `pnpm --filter @genesis/<layer> test` and `pnpm replay` and paste the output."
4. **Demand evidence.** If it says "tests pass," ask: "Show the command and its
   output." Never accept a claim without the output. (CLAUDE.md already requires this.)
5. **Independent review.** In the same session:
   > "Use the contract-reviewer subagent to review the diff against
   > plans/PHASE-0N-<layer>.md and the layer contract. Report only gaps that affect
   > correctness or the contract."
   Then have it fix real gaps.
6. **Gate.** Run `/phase-gate` (the skill) — it walks the definition-of-done
   checklist and refuses to pass until tests + contract are green.
7. **Commit.** "Commit with a message naming the phase and layer."
8. `/clear` and move to the next phase.

## The rules that keep it maintainable (so you can read the code later)

- **Determinism:** the same seed always reproduces the same world. This is what lets
  you (and the agent) catch any regression instantly with `pnpm replay`.
- **One layer = one package:** each lives in `packages/<layer>` and only talks to
  others through events + contracts. You can later rewrite any single package without
  touching the rest.
- **Tests are the contract:** if the tests pass, the layer honors its promises. You
  don't have to read every line to trust it.
- **Plans are short:** every phase has a one-page plan you approve before code is
  written. That one page is your steering wheel.

## Tips drawn from Claude Code best practices

- Use **plan mode** for anything touching more than one file. Skip it for typos.
- Keep `CLAUDE.md` short; if the agent starts ignoring rules, it's too long — prune.
- Use **subagents** for reading/exploring so your main context stays clean.
- Model choice: **Opus** when planning a phase, **Sonnet** when implementing,
  **Haiku** for the small subagent jobs (`/model` to switch).
- After two failed corrections on the same thing, `/clear` and restart with a
  sharper prompt — a clean session beats a polluted one.
- The owner's real job is **writing/approving the one-page phase plans and reading
  the test evidence.** Everything else the agent does.

## What to do when something feels off

- If a phase's tests won't go green, do **not** let the agent loosen a Law to force
  it. Ask it to explain the tradeoff and bring it to you.
- If the world stops being reproducible (`pnpm replay` diverges), that's a
  determinism bug (Law II) — highest priority, fix before anything else.
- If you don't understand a change, ask the agent: "Explain this diff like I have
  basic programming skills and have never built a game."
