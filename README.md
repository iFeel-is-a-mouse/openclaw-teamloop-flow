# team-loop — Autonomous Swarm with Self-Verifying TDD Loops

> *Each cycle triggers meta-validation: a skeptic scores alignment; low score halts. Snapshots capture intent, delta, confidence for breakpoint resume. Token budgets gate phases; coverage blocks hollow progress. Human only on value failure.*

## What

team-loop upgrades MA agent teams from human-driven waterfall delivery to **goal-driven autonomous loops**. Define a quantifiable goal with `/goal`, and the system autonomously decomposes it into KR layers, schedules agents, runs adversarial verification, and learns from failures — all without human intervention in the loop.

## Architecture

- **11-step RSF** (Roles-Steps-Flows) main loop with formal in/out validation
- **3-tier KR progression**: Foundation → Coverage → Quality
- **3-perspective adversarial verification**: reviewer (correctness) + tester (completeness) + auditor (robustness)
- **2/3 voting** for pass/fail decisions
- **14 formal data flows** — all validated: `X ∈ A.out ∩ B.in`

## Principles

1. Human defines goal, never enters loop
2. Different agents / different models / different verification methods
3. State on disk, fresh context every round
4. Infrastructure first — logging, identifiers, exception capture before business code

## Features

- **Interaction ID system** — end-to-end traceability across 9 artifact types
- **Token budget guard** — 80% model downgrade → 90% no retry → 95% pause → 100% halt
- **Coverage gates** — L1 syntax → L2 function (>80% line, >70% branch) → L3 semantic
- **Health scoring** — `100 - ERROR×5 - WARNING×2 - risk_flags×3 - stagnation - excessive_retry`
- **Exception capture** — L1 tool / L2 session / L3 loop level with watchdog + crash log
- **Separation of concerns** — logging: collection side / processing side / viewing side
- **Test death penalty** — delete wrong tests, don't patch them
- **Breakpoint resume** — StateSnapshot (t0/t1) per round with intent, delta, confidence

## Files

- `SKILL.md` — Full specification (English)
- `skill-cn.md` — Full specification (Chinese)
- `reference/` — Architecture reference materials

## Quick Start

```
/goal handler layer branch coverage 0%→≥50%, budget≤¥20
```

## Requirements

- OpenClaw runtime with MA agent team (main, coder, reviewer, tester, auditor, publicist)
- Git-tracked project with `rdf/`, `.team-loop/`, `logs/` directory structure

## License

MIT
