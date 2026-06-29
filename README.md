# team-loop — Autonomous Swarm with Self-Verifying TDD Loops

> *Each cycle triggers meta-validation: a skeptic scores alignment; low score halts. Snapshots capture intent, delta, confidence for breakpoint resume. Token budgets gate phases; coverage blocks hollow progress. Human only on value failure.*

---

## RSF Main Loop (11 steps, 14 flows)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          /goal <quantifiable target>                    │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
  ┌─────────────────────────────▼─────────────────────────────┐
  │ main (GoalAgent)                                           │
  │  seq 1  Parse → StandardGoal                               │
  └─────────────────────────────┬─────────────────────────────┘
                                │ StandardGoal                  ┌──────────────────┐
  ┌─────────────────────────────▼─────────────────────────────┐│ MemoryEntry      │
  │ main (PlannerAgent)                                        ││ (feedback loop)  │
  │  seq 2  Read RDF → 3-tier KR → TaskDAG + TokenEstimate    ││◄─────────────────┤
  │         Write ADR if tech decision needed                  ││                  │
  └──────────────┬──────────────────────────────┬─────────────┘│                  │
                 │ TaskDAG+TokEst                │ ADR           │                  │
  ┌──────────────▼──────────────────────────────┐               │                  │
  │ main (LoopController)                        │               │                  │
  │  seq 3  Pick highest-priority ready KR       │               │                  │
  │         /status + budget guard               │               │                  │
  │         → TaskUnit + StateSnapshot(t0)       │               │                  │
  └──────────────────┬──────────────────────────┘               │                  │
                     │ TaskUnit (sync)                           │                  │
  ┌──────────────────▼──────────────────────────┐               │                  │
  │ main (LoopController)                        │               │                  │
  │  seq 4  sessions_spawn → coder [300s]       │               │                  │
  └──────────────────┬──────────────────────────┘               │                  │
                     │ TaskUnit (async)                          │                  │
  ┌──────────────────▼──────────────────────────┐               │                  │
  │ coder (ExecutorAgent)                        │               │                  │
  │  seq 5  TDD 3-step: Write test → Code → Self│               │                  │
  │         → TaskResult + commit                │               │                  │
  └──────────────────┬──────────────────────────┘               │                  │
                     │ TaskResult                                │                  │
  ┌──────────────────▼──────────────────────────┐               │                  │
  │ main (Monitor)                               │               │                  │
  │  seq 6  Distribute to critics [120s]         │               │                  │
  └───┬──────────────────────────────┬──────────┘               │                  │
      │ TaskResult                   │ TaskResult               │                  │
  ┌───▼──────────────┐    ┌─────────▼───────────┐               │                  │
  │ reviewer (Critic) │    │ tester (Critic)      │               │                  │
  │  seq 7  Correct-  │    │  seq 7  Dynamic      │               │                  │
  │  ness review      │    │  verify + Coverage   │               │                  │
  │  → Verdict        │    │  → Verdict + CovRpt  │               │                  │
  └───┬──────────────┘    └─────────┬───────────┘               │                  │
      │ Verdict                     │ Verdict+CovRpt            │                  │
  ┌───┴─────────────────────────────▼──────────────────────────┐│                  │
  │ main (Monitor)                                               ││                  │
  │  seq 8  Collect verdicts → distribute to auditor [120s]    ││                  │
  └──────────────────────────────┬──────────────────────────────┘│                  │
                                 │ TaskResult+Verdict×2+CovRpt   │                  │
  ┌──────────────────────────────▼──────────────────────────────┐│                  │
  │ auditor (Critic)                                              ││                  │
  │  seq 9  13-item checklist audit → Verdict                   ││                  │
  └──────────────────────────────┬──────────────────────────────┘│                  │
                                 │ Verdict                        │                  │
  ┌──────────────────────────────▼──────────────────────────────┐│                  │
  │ main (LoopController)                                        ││                  │
  │  seq 10  2/3 voting → Decision                              ││                  │
  │          TokenReport + StateSnapshot(t1) + RoundSummary      ││                  │
  │          ┌──── retry|next_kr ────────────────────────────────┼┼──┐              │
  └──────────┼──────────────────────────────────────────────────┘│  │              │
             │                  TaskResult+Decision              │  │              │
  ┌──────────▼──────────────────────────────────────────────┐   │  │              │
  │ publicist (MemoryAgent)                                   │   │  │              │
  │  seq 11  Write RDF memory → MemoryEntry                  │   │  │              │
  └──────────────────────────────────────────────────────────┘   │  │              │
                                                                  │  │              │
  ┌──────────────────────────────────────────────────────────────┘  │              │
  │ feedback: MemoryEntry → seq 2 (next round)                     │              │
  └─────────────────────────────────────────────────────────────────┘              │
                                                                                    │
  ┌─────────────────────────────────────────────────────────────────────────────────┘
  │ retry: seq 10 → seq 3 (retry failed KR, max 3)
  │ next_kr: seq 10 → seq 3 (next ready KR)
  └──────────────────────────────────────────────────────────────────────────────────
```

---

## Architecture

- **11-step RSF** (Roles-Steps-Flows) with formal `X ∈ A.out ∩ B.in` validation
- **3-tier KR progression**: Foundation (K1) → Coverage (K2) → Quality (K3)
- **3-perspective verification**: reviewer + tester + auditor → 2/3 voting
- **14 data flows** — all programmatically verified closed
- **9 interaction ID carriers** — end-to-end traceability

## Principles

1. Human defines goal, never enters loop
2. Different agents / different models / different verification methods
3. State on disk, fresh context every round
4. Infrastructure first — logging, identifiers, exception capture before business code

## Features

| Feature | Mechanism |
|---------|-----------|
| Token budget guard | 80% model downgrade → 90% no retry → 95% pause → 100% halt |
| Coverage gates | L1 syntax → L2 function (>80% line, >70% branch) → L3 semantic |
| Health scoring | `100 - ERROR×5 - WARNING×2 - risk_flags×3 - stagnation - excessive retry` |
| Exception capture | L1 tool / L2 session / L3 loop level with watchdog + crash log |
| Breakpoint resume | StateSnapshot (t0/t1) per round: intent, delta, confidence |
| Test death penalty | Delete wrong tests; never patch them |

---

## Quick Start

```bash
# In an OpenClaw workspace with MA agent team:
/goal handler layer branch coverage 0%→≥50%, budget≤¥20
```

---

## Files

| File | Description |
|------|-------------|
| `SKILL.md` | Full English specification |
| `skill-cn.md` | Full Chinese specification |
| `README.md` | This file |
| `reference/` | Architecture references |

---

# team-loop — 自主任务循环引擎

> *每轮循环触发元验证：审查者评分对齐度；低分暂停。快照捕获意图、增量、置信度，支持断点续传。Token 预算门控各阶段；覆盖率阻止空洞进展。人仅在价值失败时介入。*

---

## RSF 主循环 (11步, 14条流)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      /goal <可量化目标>                                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
  ┌─────────────────────────────▼─────────────────────────────┐
  │ main (GoalAgent)         seq 1  解析目标 → StandardGoal    │
  └─────────────────────────────┬─────────────────────────────┘
                                │ StandardGoal         ┌──────────────────┐
  ┌─────────────────────────────▼──────────────────────┐│ MemoryEntry      │
  │ main (PlannerAgent)         seq 2  读RDF → 三层KR  ││ (反馈流)          │
  │   拆解 → TaskDAG + TokenEstimate 有选型 → 写ADR    ││◄─────────────────┤
  └─────────────────────────────┬──────────────────────┘│                  │
                                │                        │                  │
  ┌─────────────────────────────▼──────────────────────┐│                  │
  │ main (LoopController)       seq 3  取就绪KR         ││                  │
  │   /status + 预算守卫 → TaskUnit + StateSnapshot(t0) ││                  │
  └─────────────────────────────┬──────────────────────┘│                  │
                                │ TaskUnit               │                  │
  ┌─────────────────────────────▼──────────────────────┐│                  │
  │ main (LoopController)       seq 4  下发coder [300s] ││                  │
  └─────────────────────────────┬──────────────────────┘│                  │
                                │ TaskUnit               │                  │
  ┌─────────────────────────────▼──────────────────────┐│                  │
  │ coder (ExecutorAgent)       seq 5  TDD三步          ││                  │
  │   写测试 → 写实现 → 自检 → TaskResult + commit      ││                  │
  └─────────────────────────────┬──────────────────────┘│                  │
                                │ TaskResult              │                  │
  ┌─────────────────────────────▼──────────────────────┐│                  │
  │ main (Monitor)              seq 6  收集 → 分发critic││                  │
  └───┬─────────────────────┬──────────────────────────┘│                  │
      │ TaskResult          │ TaskResult                 │                  │
  ┌───▼──────────────┐ ┌────▼──────────────┐            │                  │
  │ reviewer (Critic) │ │ tester (Critic)   │            │                  │
  │  seq 7  正确性审查│ │  seq 7  动态验证  │            │                  │
  │  → Verdict        │ │  → Verdict+CovRpt │            │                  │
  └───┬──────────────┘ └────┬──────────────┘            │                  │
      │                     │                            │                  │
  ┌───┴─────────────────────▼────────────────────────────┐│                  │
  │ main (Monitor)              seq 8  收判决 → 分发auditor│                  │
  └────────────────────────────┬─────────────────────────┘│                  │
                               │ TaskResult+Verdict×2+Cov │                  │
  ┌────────────────────────────▼─────────────────────────┐│                  │
  │ auditor (Critic)            seq 9  13项checklist审计  ││                  │
  │   → Verdict                                          ││                  │
  └────────────────────────────┬─────────────────────────┘│                  │
                               │ Verdict                   │                  │
  ┌────────────────────────────▼─────────────────────────┐│                  │
  │ main (LoopController)       seq 10  2/3裁决          ││                  │
  │   → Decision + TokenReport + StateSnapshot(t1)       ││                  │
  │   ┌─── retry|next_kr ────────────────────────────────┼┼──┐              │
  └───┼──────────────────────────────────────────────────┘│  │              │
      │               TaskResult+Decision                  │  │              │
  ┌───▼──────────────────────────────────────────────┐    │  │              │
  │ publicist (MemoryAgent)   seq 11  沉淀RDF记忆     │    │  │              │
  │   → MemoryEntry                                    │    │  │              │
  └────────────────────────────────────────────────────┘    │  │              │
                                                            │  │              │
  ┌────────────────────────────────────────────────────────┘  │              │
  │ 反馈: MemoryEntry → seq 2 (下一轮PlannerAgent查询)         │              │
  └───────────────────────────────────────────────────────────┘              │
                                                                              │
  ┌───────────────────────────────────────────────────────────────────────────┘
  │ retry: seq 10 → seq 3 (KR失败重试，最多3次)
  │ next_kr: seq 10 → seq 3 (取下一就绪KR)
  └────────────────────────────────────────────────────────────────────────────
```

---

## 核心原则

1. 人只定义目标，不进循环
2. 不同 agent / 不同模型 / 不同验证手段
3. 状态在磁盘，每轮 fresh context
4. 基础设施先行 — 日志、编号、异常捕捉在业务代码之前到位

## 功能一览

| 功能 | 机制 |
|------|------|
| Token 预算守卫 | 80% 降级 → 90% 不重试 → 95% 暂停 → 100% 终止 |
| 覆盖率门禁 | L1 语法 → L2 功能(>80%行, >70%分支) → L3 语义 |
| 健康度评分 | `100 - ERROR×5 - WARNING×2 - risk_flags×3 - 停滞 - 过度重试` |
| 异常捕捉 | L1 工具级 / L2 Session级 / L3 循环级 + 看门狗 + 兜底日志 |
| 断点续传 | 每轮 StateSnapshot(t0/t1)：意图、增量、置信度 |
| 测试死刑 | 错误测试 → 删 + 重写，禁止修补 |

---

## 快速开始

```bash
# 在配置了 MA agent 团队的 OpenClaw 工作区：
/goal handler层分支覆盖率 0%→≥50%，预算≤¥20
```

---

## 文件

| 文件 | 说明 |
|------|------|
| `SKILL.md` | 英文完整规格 |
| `skill-cn.md` | 中文完整规格 |
| `README.md` | 本文件 |
| `reference/` | 架构参考资料 |

---

## License

MIT
