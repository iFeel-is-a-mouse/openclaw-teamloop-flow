# team-loop — Autonomous Swarm with Self-Verifying TDD Loops

> *Each cycle triggers meta-validation: a skeptic scores alignment; low score halts. Snapshots capture intent, delta, confidence for breakpoint resume. Token budgets gate phases; coverage blocks hollow progress. Human only on value failure.*

---

## RSF Main Loop (11 steps, 14 flows)

```
/goal ──────────────────────────────────────────────────────────────────────┐
                                                                            │
  ┌─ seq1  main(GoalAgent)    Parse → StandardGoal                          │
  │         f1 sync ──────────────────────────────┐                         │
  ├─ seq2  main(Planner)      Read RDF → 3-tier KR│→ TaskDAG+TokEst + ADR  │
  │         f2 sync ────────────────────────┐     │                         │
  ├─ seq3  main(Controller)   Pick KR ──────┤───→ TaskUnit + Snap(t0)      │
  │         /status+budget guard             │     │                         │
  │         f3 sync ─────────────────────────┤     │                         │
  ├─ seq4  main(Controller)   Spawn coder [300s]   │                         │
  │         f4 async ────────────────────────┼───→ coder                     │
  ├─ seq5  coder(Executor)    TDD 3-step ────┼───→ TaskResult+commit        │
  │         f5 async ←───────────────────────┘     │                         │
  ├─ seq6  main(Monitor)      Distribute [120s] ───┼──→ reviewer + tester   │
  │         f6→reviewer  f7→tester                  │     │                  │
  ├─ seq7  reviewer(Critic)   Correctness ─────────┼───→ Verdict            │
  │         tester(Critic)     Dynamic+Cov ─────────┼───→ Verdict+CovRpt    │
  │         f8,f9 async ←───────────────────────────┘     │                  │
  ├─ seq8  main(Monitor)      Collect → spawn auditor ────┼──→ auditor       │
  │         f10 async ──────────────────────────────────→ │                  │
  ├─ seq9  auditor(Critic)    13-item checklist ──────────┼───→ Verdict      │
  │         f11 async ←───────────────────────────────────┘                  │
  ├─ seq10 main(Controller)   2/3 vote → Decision                          │
  │         → TokenReport + Snap(t1) + RoundSummary                         │
  │         f12 async ──────────────────────→ publicist                     │
  ├─ seq11 publicist(Memory)  RDF memory → MemoryEntry                      │
  │         f13 async ←── feedback to seq2 ──────────────────────────────────┤
  │                                                                          │
  │         fR: retry|next_kr ──→ seq3 ───────────────────────────────────┐ │
  └────────────────────────────────────────────────────────────────────────┘ │
                                                                             │
  Stop conditions: all-KR | FATAL(3x fail) | timeout | block | BUDGET |     │
  max_rounds | stagnation(3x) | MANUAL_STOP | escape | watchdog ◄───────────┘
```

---

## Architecture

- **11-step RSF** with formal `X ∈ A.out ∩ B.in` validation — all 14 flows programmatically verified closed
- **3-tier KR**: Foundation (K1) → Coverage (K2) → Quality (K3)
- **3-perspective verification**: reviewer + tester + auditor → 2/3 voting
- **9 interaction ID carriers** — end-to-end traceability

## Principles

1. Human defines goal, never enters loop
2. Different agents / different models / different verification methods
3. State on disk, fresh context every round
4. Infrastructure first — logging, IDs, exception capture before business code

## Features

| Feature | Mechanism |
|---------|-----------|
| Token budget guard | 80% downgrade → 90% no retry → 95% pause → 100% halt |
| Coverage gates | L1 syntax → L2 func (>80% line, >70% branch) → L3 semantic |
| Health scoring | `100 - ERROR×5 - WARNING×2 - risk_flags×3 - stagnation - excess retry` |
| Exception capture | L1 tool / L2 session / L3 loop + watchdog + crash log |
| Breakpoint resume | StateSnapshot(t0/t1): intent, delta, confidence per round |
| Test death penalty | Delete wrong tests; never patch them |

---

## Quick Start

```bash
/goal handler layer branch coverage 0%→≥50%, budget≤¥20
```

---

## Files

| File | Description |
|------|-------------|
| `SKILL.md` | Full English specification |
| `skill-cn.md` | Full Chinese specification |
| `README.md` | This file |

---

# team-loop — 自主任务循环引擎

> *每轮触发元验证：审查者评分对齐；低分暂停。快照捕获意图、增量、置信度支持断点续传。Token 预算门控各阶段；覆盖率阻止空洞进展。人仅在价值失败时介入。*

---

## RSF 主循环 (11步 14流)

```
/goal ──────────────────────────────────────────────────────────────────────┐
                                                                            │
  ┌─ seq1  main(GoalAgent)    解析 → StandardGoal                           │
  │         f1 sync ──────────────────────────────┐                          │
  ├─ seq2  main(Planner)      读RDF → 三层KR ────│→ TaskDAG+TokEst + ADR   │
  │         f2 sync ────────────────────────┐     │                          │
  ├─ seq3  main(Controller)   取就绪KR ─────┤───→ TaskUnit + Snap(t0)       │
  │         /status+预算守卫                  │     │                          │
  │         f3 sync ─────────────────────────┤     │                          │
  ├─ seq4  main(Controller)   下发coder [300s]     │                          │
  │         f4 async ────────────────────────┼───→ coder                      │
  ├─ seq5  coder(Executor)    TDD三步 ──────┼───→ TaskResult+commit          │
  │         f5 async ←──────────────────────┘     │                          │
  ├─ seq6  main(Monitor)      分发 [120s] ────────┼──→ reviewer + tester     │
  │         f6→reviewer  f7→tester                  │     │                   │
  ├─ seq7  reviewer(Critic)   正确性审查 ─────────┼───→ Verdict               │
  │         tester(Critic)     动态验证+Cov ──────┼───→ Verdict+CovRpt        │
  │         f8,f9 async ←─────────────────────────┘     │                    │
  ├─ seq8  main(Monitor)      收→发auditor ────────────┼──→ auditor          │
  │         f10 async ────────────────────────────────→ │                    │
  ├─ seq9  auditor(Critic)    13项checklist ───────────┼───→ Verdict          │
  │         f11 async ←────────────────────────────────┘                     │
  ├─ seq10 main(Controller)   2/3裁决 → Decision                              │
  │         → TokenReport + Snap(t1) + RoundSummary                           │
  │         f12 async ──────────────────────→ publicist                       │
  ├─ seq11 publicist(Memory)  沉淀RDF → MemoryEntry                           │
  │         f13 async ←── 反馈至seq2 ────────────────────────────────────────┤
  │                                                                           │
  │         fR: retry|next_kr ──→ seq3 ────────────────────────────────────┐ │
  └─────────────────────────────────────────────────────────────────────────┘ │
                                                                              │
  停止条件: all-KR | FATAL(连3失败) | 超时 | 阻塞 | BUDGET | max_rounds |     │
  stagnation(连3轮) | MANUAL_STOP | 逃逸 | watchdog ◄────────────────────────┘
```

---

## 核心原则

1. 人只定义目标，不进循环
2. 不同 agent / 不同模型 / 不同验证手段
3. 状态在磁盘，每轮 fresh context
4. 基础设施先行 — 日志、编号、异常捕捉在业务代码之前

## 功能

| 功能 | 机制 |
|------|------|
| Token 预算守卫 | 80% 降级 → 90% 不重试 → 95% 暂停 → 100% 终止 |
| 覆盖率门禁 | L1 语法 → L2 功能(>80%行, >70%分支) → L3 语义 |
| 健康度评分 | `100 - ERROR×5 - WARNING×2 - risk_flags×3 - 停滞 - 过度重试` |
| 异常捕捉 | L1 工具 / L2 Session / L3 循环 + 看门狗 + 兜底日志 |
| 断点续传 | StateSnapshot(t0/t1)：意图、增量、置信度 |
| 测试死刑 | 错误测试 → 删+重写，禁止修补 |

---

## 快速开始

```bash
/goal handler层分支覆盖率 0%→≥50%，预算≤¥20
```

---

## License

MIT
