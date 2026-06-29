
# team-loop 项目立项书与可行性分析

> 版本：v1.45 | 日期：2026-06-28 | 作者：iFeel & iClaw

---

## 一、项目概述

### 1.1 项目名称

**team-loop** — 基于 MA（Multi-Agent）体系的自主任务循环引擎

### 1.2 一句话定义

将 MA 现有的 agent 团队（main/coder/reviewer/tester/auditor/publicist）从「人驱动的瀑布交付」升级为「目标驱动的自主循环」，让系统在无人值守下持续执行任务、对抗验证、从失败中学习。

### 1.3 项目来源

本项目的认知基础来自两条线的交叉验证：

- **实践线**：debugloop 在 pyOV 项目上暴露的信息不对称问题 —— LLM 有零件缺装配图，3 个生成测试全部在 setup 阶段崩溃，覆盖率零提升
- **理论线**：Loop Engineering 框架（Addy Osmani / Boris Cherny / Peter Steinberger）—— Automations、Worktrees、Skills、Sub-agents、Memory 五构建块 + 对抗验证

两条线汇聚到一个结论：**AI agent 系统的瓶颈不是 LLM 不够聪明，而是缺少正确的上下文、独立的验证、和跨轮次的学习。**

---

## 二、核心洞察与方法论

### 2.1 三个核心洞察

| # | 洞察 | 来源 |
|---|------|------|
| 1 | **不翻译，喂原材** — LLM 的代码理解能力远超任何手写解析器，不要 AST 中间翻译层 | 对 debugloop 分析方案的反思 |
| 2 | **maker/checker 必须分离** — 写代码的 agent 和检查代码的 agent 必须是不同实体 | Loop Engineering CriticAgent + MA 已有 coder/reviewer 分离 |
| 3 | **知识应常驻、可按需查询** — 项目上下文不应每次都灌入 prompt，而应作为可查询的外部记忆 | Loop Engineering Memory + 自研 Knowledge Server |

### 2.2 方法论原则

```
原则 1：喂原始文件，不翻译
原则 2：知识常驻伺服，不灌 prompt
原则 3：maker/checker 分离，3 视角对抗验证（correctness/completeness/robustness）
原则 4：Ralph 技法 — fresh context + 原子化工作单元 + 状态在磁盘不在窗口
原则 5：从失败中学习 — 记忆闭环，同样错误不犯第二次
原则 6：可验证的停止条件 — "覆盖率 ≥ 50%" 而非 "差不多了"
```

---

### 2.3 设计约束

以上 6 条方法论原则落地到以下机制：

- 原则1(喂原材) + 原则2(知识常驻) → Knowledge Server (§3.2.1)
- 原则3(maker/checker分离) → 三视角对抗验证 (§3.6)
- 原则4(Ralph技法) → while 循环 + fresh context + STATE.md (§2.5)
- 原则5(从失败学习) → Memory + PlannerAgent 反馈环 (§3.8.4 f16)
- 原则6(可验证停止) → 11 条终止规则 (§3.4)

---

### 2.4 停止条件

team-loop 的终止与循环退出必须由可程序化验证的条件驱动，不依赖人类的主观判断。详细的 7 条终止规则、优先级顺序和终止后行为定义在 §3.4 停止条件（可验证的合同式终止）中。

核心原则：**停止条件是合同，不是感觉。** 每轮结束时 LoopController 按 check_order 优先级逐条判定，命中第一个即终止。

---

### 2.5 最高层契约

team-loop 的全部行为收敛为以下标准文档。类比 Ralph 原型——看起来太简单，但契约全在里面。

#### 2.5.1 契约

```
# team-loop 契约
/goal "把 ${module} 覆盖率提到 ${target}%，预算 ${budget}"

# 以下全自动，人类退出
while ! grep -q "终止" .team-loop/STATE.md; do   ← STATE.md 含项目累计费用 + 当前loop费用 + 状态

  planner "读 memory 和 项目知识。
           拆为 K1/K2/K3 三层 KR。
           然后停。"

  for task in 就绪的KR; do

    main 下发 ${task} →
    coder     "按 ${task} 写代码或测试。
               产出 TaskResult + reasoning 自述。
               然后停。"

    main 收集结果 → 分发三视角 →
    reviewer  "读 ${task} 的 diff。静态审查正确性。
               不运行代码。投 passed/failed。然后停。"
    tester    "跑 ${task} 的测试。量覆盖率。
               环境隔离。投 passed/failed。然后停。"
    auditor   "审 reviewer+tester 的报告。过 checklist。
               不执行代码。投 passed/failed。然后停。"

    if [ 3票中 >= 2票 passed ]; then
      publicist "记 memory: ${task} 成功。然后停。"
    else
      publicist "记 memory: ${task} 失败→${否决原因}。然后停。"
    fi

  done

  if [ K1基础层 连续3轮全失败 ]; then
    echo "FATAL" >> .team-loop/STATE.md
  fi

done

publicist "产出最终报告。然后停。"
main 通知人类: ${结果}
```

#### 2.5.2 与 Ralph 的四项对位

| Ralph | team-loop |
|-------|-----------|
| `while ! grep -q "ALL TASKS DONE" STATUS.md` | `while ! grep -q "终止" .team-loop/STATE.md` |
| `claude -p "Read PLAN.md and STATUS.md..."` | main spawn 6 个 agent，每个给明确指令，结尾都是 "然后停" |
| `commit on success` | publicist 写 memory 持久化，CR 记录变更 |
| `Then stop.` | 每个 agent 指令结尾 "然后停。" — fresh context，原子化 |

#### 2.5.3 四条不可妥协的线

1. **人只定义目标，不进循环。** /goal 设定 O 后退出，系统自主运行至终止条件触发。人类可通过创建 `.team-loop/STOP` 文件安全中断循环。
2. **写代码的和检查代码的永远是不同 agent、不同模型、不同验证手段。** coder 不审自己代码，reviewer 不运行代码，auditor 不执行代码。
3. **状态在磁盘，不在上下文里。** memory + STATE + coverage 报告 + CR 记录全部持久化，每轮 fresh isolated context 启动。
4. **停止条件可程序化验证，不靠"感觉差不多了"。** `grep -q "终止" STATE` — 7 条终止规则全为可计算表达式。

### 2.6 TDD 在 team-loop 中的角色

team-loop 的核心产出是测试，因此 TDD（测试驱动开发）不是方法论选项，而是内置约束。

#### 2.6.1 coder 的执行规则

coder 按 TaskUnit 执行 KR 时，必须遵循 TDD 三步：

```
1. 写测试 → 验证测试本身正确（语法、导入、可运行）
2. 如果 KR 要求实现代码 → 写最少代码让测试通过
3. 自检：pytest 全通过 + reasoning 自述做了什么
```

**约束：** coder 不跳过步骤 1。即使 KR 是"补测已有代码"，也必须先确保所写测试可被 pytest 收集和运行——这正是 K1 基础层的意义。

#### 2.6.2 与 KR 三层模型的映射

| TDD 步骤 | KR 层 | 验证者 | 失败后果 |
|---------|-------|--------|---------|
| 写测试，验证可运行 | K1 基础层 | tester | 连续 3 轮失败 → FATAL |
| 测试覆盖目标分支 | K2 覆盖层 | tester (覆盖率) | 单 KR 失败 → 重试 → 跳过 |
| 已有测试全通过 | K3 质量层 | tester (回归) | 回滚本轮，记录根因 |

#### 2.6.3 TDD 循环与 Loop 循环的嵌套

```
Loop 循环（外层）
  └── 每轮迭代
        └── KR 执行（内层 TDD 循环）
              ├── coder 写测试 → tester 验证可运行
              ├── coder 写实现 → tester 验证通过+覆盖率
              └── tester 回归验证 → 全通过
```

#### 2.6.4 测试先行的具体规则

coder 收到 TaskUnit 后，必须按以下顺序执行（不可跳过任何步骤）：

**Step 1: 写测试，验证可运行**

```
1. 读取 TaskUnit.context 中的 KS 上下文（ctx 结构、已有范例、import 路径）
2. 编写测试文件 tests/test_*.py，遵循 AAA 模式：
   - Arrange: 构造测试数据、初始化依赖（按 runtime-ctx.ttl 的 ctx 结构）
   - Act: 调用被测函数/发送 HTTP 请求
   - Assert: 验证返回值/响应码/副作用
3. 自运行 pytest 验证：
   - pytest 能收集到此测试（不被 skip/过滤）
   - 测试能开始执行（无 import 错误、setup 崩溃）
   - 测试断言有意义（不是 assert True，不是只调用函数不验证结果）
4. 如果步骤 3 失败 → 修复测试后重试，不进入步骤 2
5. 如果步骤 3 通过 → 进入步骤 2
```

**Step 2: 如果需要实现代码，写最少代码让测试通过**

```
1. 仅在 KR 要求实现代码时执行（如"修复 handle_tree 的边界条件"）
2. 写最少代码让步骤 1 的测试通过
3. 不修改测试文件以外的不相关代码
4. 如果 KR 是"补测已有代码" → 跳过此步骤，当前代码即为被测对象
```

**Step 3: 自检，产出 TaskResult**

```
1. 运行全量测试套件（不限于本次新增/修改的测试）
2. 确认无回归（已有测试全部通过）
3. 确认新测试通过
4. 填写 reasoning 自述：
   - 为哪个模块的哪个函数编写了测试
   - 测试方法（HTTP请求/直接调用/单元测试）
   - 覆盖了哪些分支/路径
   - 遇到的问题及解决方式（无则写"无"）
5. 产出 TaskResult 文件：
   - output: 代码 diff 或新增文件路径
   - status: "success"
   - files_changed: [变更的文件列表]
   - reasoning: 以上自述
```

#### 2.6.5 coder → tester 的交付契约

coder 完成自检后，通过 main 将 TaskResult 传递给 tester。tester 收件后执行以下验证：

```
1. 环境隔离验证
   - 在独立目录运行（不依赖 coder 的本地环境副作用）
   - 仅安装 pyproject.toml 或 requirements.txt 声明的依赖
   - 无额外 pip install

2. 测试可运行性验证
   - pytest --collect-only 确认测试能被收集
   - 首次运行确认无 import/setup 错误
   - 如果 setup 崩溃 → Verdict: FAILED, "测试无法运行：{错误信息}"

3. 覆盖率验证
   - pytest --cov=target_module --cov-report=json
   - 读取 coverage.json，提取目标文件的 covered_branches
   - 对比基线，计算 delta
   - 如果 delta ≤ 0 → Verdict: FAILED, "覆盖率无增量"

4. 断言有效性验证
   - 检查测试是否真正验证了预期行为
   - 如果所有测试都是 assert True 或只调用不验证 → Verdict: FAILED, "断言无效"

5. flaky 检测
   - 同一测试连续运行 3 次
   - 如果结果不一致 → Verdict: FAILED, "测试不稳定：{通过/失败次数}"

6. 回归安全
   - 运行全量测试套件
   - 如有已有测试失败 → Verdict: FAILED, "回归失败：{失败测试列表}"

7. 产出 Verdict + CoverageReport
   - 以上 6 项全部通过 → passed: true
   - 任一项失败 → passed: false，issues 列出具体失败项
```

**tester 不检查的边界：**
- 不检查代码逻辑是否正确 → 那是 reviewer 的职责
- 不检查安全性 → 那是 auditor 的职责
- 不修改源文件 → 只读和执行

## 三、架构设计

### 3.1 Agent 角色映射（复用 MA 现有 agent）

#### 3.1.1 角色分配

```
loopengine 角色          MA agent           职责
─────────────────────   ────────────────    ──────────────────
GoalAgent + Planner      main               目标解析 + 任务拆解 + 调度决策
ExecutorAgent            coder               执行任务（写代码/测试/修复）
CriticAgent (correct)    reviewer           正确性验证 — code review
CriticAgent (complete)   tester             完整性验证 — 覆盖率 + 断言有效性 + 环境隔离
CriticAgent (robust)     auditor            健壮性审计 — checklist 审查 + 流程合规（永不执行代码）
MemoryAgent              publicist          知识沉淀 + 文档记录
MonitorAgent             main               进度跟踪 + 异常告警
LoopController           main               终止条件判断 + 循环调度
```

**零新 agent，全部复用现有 MA 配置。**

#### 3.1.2 每个 agent 的输入输出契约与工具约束

| Agent | 输入 | 输出 | 可用工具 | 不可用工具 | 约束 |
|-------|------|------|---------|-----------|------|
| **main** (GoalAgent) | `raw_goal: string` | `StandardGoal` | LLM | exec, write | 调用LLM；不查询项目知识，不访问 KS |
| **main** (PlannerAgent) | `StandardGoal` + `MemoryEntry[]` + `KSContext` | `TaskDAG` | LLM, read | exec, write | 调用LLM；必须查询 KS 获取项目上下文，必须查询 Memory 获取历史 |
| **main** (LoopController) | `TaskDAG` + `Verdict[]` | `Decision` (next/retry/skip/terminate) | — | LLM, exec, write | 纯程序化，不调用 LLM |
| **main** (Monitor) | `Task[]` + `Log[]` | `RoundSummary` | — | LLM, exec, write | 纯程序化，聚合指标。告警规则：failure_rate>50%触发、in_progress>3触发(可能死锁)、5轮后completion<30%触发(进展缓慢) |
| **coder** | `TaskUnit` | `TaskResult` | LLM, read, write, exec | — | 按需查询 KS；不审查自己代码 |
| **reviewer** | `TaskResult` | `Verdict` (correctness) | read | exec, write | 仅静态读代码，不运行代码，不修改文件 |
| **tester** | `TaskResult` | `Verdict` (completeness) + `CoverageReport` | read, exec | write (源文件) | 必须运行代码，负责环境隔离 |
| **auditor** | `TaskResult` + 审查记录 | `Verdict` (robustness) | read | exec, write | 永不执行代码，永不修改文件，审查的是审查者有没有审查到位 |
| **publicist** | `TaskResult` + `Decision` | `MemoryEntry` | write | exec | 只记录不执行，结构化 markdown |

#### 3.1.3 agent 独立性红线

| Agent | 红线 | 原因 |
|-------|------|------|
| coder | 不审查自己代码 | 自评偏见 |
| reviewer | 不运行代码 | 运行=执行=参与=失去独立 |
| tester | 不审查代码逻辑 | 那是 reviewer 的事 |
| auditor | 永不执行代码，永不修改文件 | 执行=参与=自己评自己 |

### 3.2 新增组件（仅 2 个）

#### 3.2.1 Knowledge Server（KS）

**定位：** 项目知识的增删改查工具。Python 库引入（`import ks`），知识存储在项目本地 `rdf/` 目录，基于 rdflib + SPARQL。

**接口概要：**
```python
from ks import KnowledgeServer
ks = KnowledgeServer(project_path="/path/to/project")
ks.index()                                                          # 扫描源码生成 rdf/ast/*.ttl
context = ks.query(module="handlers.py", keywords=["ctx", "fixture"])  # SPARQL 查询
ks.insert(graph="memory", triple=(s, p, o))                        # 写入三元组
```

**实现项目：** `projects/pyrdf-knowledge` — RDF schema 定义、Store API、Query API、Indexer 全部位于该项目中。team-loop 仅作为调用方使用 KS 公开接口，不包含 KS 实现细节。

#### 3.2.2 循环调度模式（main 新增人格）

**触发方式：** 人类输入 `/goal <目标描述>` → main 切换至 team-loop 人格。

**调度规格：**

```json
{
  "trigger": "/goal <raw_goal>",
  "mode": "team-loop",
  "entry": {
    "phase": "goal_standardization",
    "agent": "main",
    "action": "parse_raw_goal_to_standard_goal"
  },
  "loop": {
    "start": "query_ks_and_plan",
    "body": ["dispatch_task", "collect_verdicts", "judge_and_record"],
    "end": "check_termination",
    "feedback": "if not terminated, back to start with memory context"
  },
  "exit": {
    "on_success": "report SUCCESS with RoundSummary",
    "on_max_rounds": "report MAX_ROUNDS with progress",
    "on_budget": "report BUDGET with token summary",
    "on_fatal": "report FATAL with diagnostic record"
  }
}
```

**与团队研发模式隔离：** `/goal` 是新增触发路径，不影响 `团队研发` 关键词触发的 12 阶段瀑布流程。同一 main agent，两套运行模式，按触发关键词分流。

### 3.3 运行流程

详见 §3.8.4 主循环流程定义。

### 3.4 停止条件（可验证的合同式终止）

停止条件在 §3.8.4 主循环 seq 8 的 main 任务中被检查，优先级按 check_order 顺序判定，命中第一个即终止。

```json
{
  "termination": {
    "description": "LoopController在每轮seq8执行的终止判断规则",
    "check_order": ["all_kr_completed", "basic_layer_fatal", "all_kr_blocked", "critic_unavailable", "budget_exhausted", "max_rounds", "stagnation", "disk_full", "manual_stop"],
    "rules": [
      {
        "id": "all_kr_completed",
        "condition": "len(pending_tasks) == 0 AND len(in_progress_tasks) == 0",
        "result": "SUCCESS",
        "description": "所有KR完成，目标达成 [正向终止]"
      },
      {
        "id": "basic_layer_fatal",
        "condition": "consecutive_basic_layer_failures >= 3",
        "result": "FATAL",
        "description": "基础层KR连续3轮失败，项目infrastructure不可用 [异常终止]",
        "diagnostic": "必须包含：最后失败KR的task_id、critic否决原因、knowledge_server可达性"
      },
      {
        "id": "budget_exhausted",
        "condition": "tokens_consumed + estimated_next_round_tokens > budget_limit",
        "result": "BUDGET",
        "description": "预算不足以支撑下一轮 [保护终止]",
        "diagnostic": "必须包含：已消耗token数、预算上限、下一轮预估消耗"
      },
      {
        "id": "max_rounds",
        "condition": "current_round >= max_rounds",
        "result": "MAX_ROUNDS",
        "description": "达到最大轮次 [保护终止]",
        "diagnostic": "必须包含：已完成轮次数、KR完成率、覆盖率变化趋势"
      },
      {
        "id": "stagnation",
        "condition": "coverage_delta == 0 AND consecutive_stagnant_rounds >= 3",
        "result": "STAGNATION",
        "description": "连续3轮覆盖率无提升 [效率终止]",
        "diagnostic": "必须包含：各轮覆盖率值、停滞开始轮次、未覆盖分支数"
      },
      {
        "id": "ks_unreachable",
        "condition": "knowledge_server_ping_failed >= 3",
        "result": "FATAL",
        "description": "Knowledge Server 不可达 [异常终止]"
      },
      {
        "id": "agent_timeout",
        "condition": "any agent response time > timeout_limit AND retry_count >= max_retries",
        "result": "FATAL",
        "description": "agent超时且重试耗尽 [异常终止]"
      },
      {
        "id": "all_kr_blocked",
        "condition": "len(pending_tasks) > 0 AND all pending tasks have dependencies that are permanently unsatisfiable",
        "result": "FATAL",
        "description": "所有剩余KR的依赖永久不可满足 [DAG软死锁]",
        "diagnostic": "必须包含：被跳过的KR列表、因此阻塞的下游KR列表"
      },
      {
        "id": "critic_unavailable",
        "condition": "any critic agent sessions_spawn failed AND retry_count >= 3",
        "result": "FATAL",
        "description": "critic agent无法创建，可能配置错误或模型不可用 [异常终止]",
        "diagnostic": "必须包含：失败的critic角色名、spawn错误信息"
      },
      {
        "id": "disk_full",
        "condition": "STATE.md or any log file write failed due to ENOSPC",
        "result": "FATAL",
        "description": "磁盘空间不足，无法写入状态或日志文件 [异常终止]"
      },
      {
        "id": "manual_stop",
        "condition": "file '.team-loop/STOP' exists",
        "result": "MANUAL_STOP",
        "description": "人类主动中断循环 [人工终止]"
      }
    ]
  }
}
```

**终止后行为：**
- SUCCESS → publicist 产出最终报告 → main 通知人类
- MAX_ROUNDS / BUDGET / STAGNATION → publicist 产出进度报告 → main 通知人类，附带建议（增加预算/增加轮次/调整目标）
- FATAL → 立即告警 → publicist 产出 MDR（最低诊断记录，见 §3.7.1.2）→ main 通知人类，不尝试自动恢复

**STATE.md 格式（while 循环 grep 此文件）：**

```markdown
# .team-loop/STATE.md

## 状态
running

## 项目累计费用
- 总 token: 1,234,567
- 总轮次: 42
- 总 loop 次数: 7
- 总耗时: 3h 21m

## 当前 loop
- loop_id: manual-20260628T140000-a3f2-L
- 开始时间: 2026-06-28T14:00:00Z
- 已执行轮次: 5
- 当前 token: 234,567
- 当前耗时: 23m

## 最近一次执行
- loop_id: manual-20260628T090000-b7e1-L
- token: 156,789
- 轮次: 3
- 结果: SUCCESS
- 耗时: 12m
```

**费用追踪规则：**
- **项目累计**：所有历史 loop 的汇总，publicist 在每轮结束时累加更新
- **当前 loop**：本次 /goal 启动后至今，Monitor 每轮更新
- **最近一次执行**：上一次 /goal 的汇总，供 Planner 评估"下次大概花多少"
- **终止判定不变**：`grep -q "终止" STATE.md` — 仅当状态写入 FATAL/MAX_ROUNDS/BUDGET/STAGNATION/MANUAL_STOP/SUCCESS 时触发

---

### 3.5 GoalAgent (O) ↔ PlannerAgent (KR)：OKR 分解模型

#### 3.5.1 映射关系

| 角色 | OKR 维度 | 职责 | 产出 |
|------|---------|------|------|
| **GoalAgent** | **O — Objective** | 定义目标方向与可验证的成功标准 | StandardGoal（不可妥协的合同） |
| **PlannerAgent** | **KR — Key Results** | 拆解为可独立执行、可独立验证的关键结果 | Task[] + DAG（到达目标的路径） |

GoalAgent 与 PlannerAgent 的协作流程详见 §3.8.4 主循环流程定义（seq 1-2：目标解析 → 规划拆解）。O/KR 映射关系和各 Agent 设计约束保留如下。

#### 3.5.2 软件工程 OKR vs 传统管理 OKR — 根本分歧

传统 OKR（Google/Intel 方法论）鼓励设 "stretch goals"：目标设得高，达成 70-80% 即为优秀。未达成不是失败，而是"我们努力过了"。

**软件工程 OKR 完全不同：**

| 维度 | 传统 OKR | 软件工程 OKR（本系统） |
|------|---------|---------------------|
| 目标设定 | 设高，激发突破 | 设准，要求 99% 可实现 |
| 达成率 | 70-80% = 优秀 | 99% = 合格，低于此 = 失败 |
| 超额完成 | 惊喜，值得庆祝 | **惊吓，需要复盘** — 系统做了未预期的事 |
| 失败态度 | "我们学到了" | "KR 拆解出错了，必须修正" |
| 验证方式 | 季度回顾，人判断 | 实时验证，CriticAgent 程序化判断 |

**为什么软件工程中"超额完成"是惊吓：**

1. **自主循环无人值守。** CriticAgent 依赖精确定义的成功标准做 2/3 通过判断。KR 模糊 → loop 永不停机或过早终止。
2. **超出规划 = 偏离规划。** 如果 planner 设 KR "生成 3 个测试"，实际生成 10 个，说明 planner 低估了 coder 能力，或 coder 偏离了 DAG 依赖链。
3. **无监督放大。** "超额"在无监督循环中可能意味着系统越权——改了不该改的文件、调了不该调的 API、消耗了未预算的 token。

**示例：同一个目标的两种拆解**

```
❌ 传统 OKR（不可用于自主系统）：
  O:  大幅提升测试覆盖率
  KR1: 覆盖率提升 30%（stretch goal, 20% 即满意）
  KR2: 测试通过率达到 90%（70% 即可接受）
  → CriticAgent 无法判断：20% 通过还是失败？模糊。

✅ 软件工程 OKR（team-loop 采用）：
  O:  handler 层分支覆盖率从基线 0% 提升至 ≥50%
  KR1: 所有 handler 文件至少有 1 个测试可运行    ← 100% 必须达成
  KR2: 每个 handler 文件覆盖率 ≥50%               ← 100% 必须达成
  KR3: 已有 743 个测试全通过，无回归               ← 100% 必须达成
  → CriticAgent 逐项判断：通过/失败，无灰色地带。
```

#### 3.5.3 GoalAgent (O) 设计约束

GoalAgent 产出的 Objective 必须同时满足 4 条约束：

| # | 约束 | 反例 | 正例 |
|---|------|------|------|
| 1 | **可量化** | "提高代码质量" | "handler 层分支覆盖率 ≥50%" |
| 2 | **有边界** | 不声明不做什么 | "不修改源码；不引入新依赖；预算 ≤¥20" |
| 3 | **可验证** | "代码更健壮" | "新测试连续 3 次运行均通过" |
| 4 | **99% 可实现** | 设 85% 覆盖率但基线仅 0% | 设 50% 覆盖率，已知范例测试可达更高 |

GoalAgent 的 System Prompt 核心指令：

```
你是目标标准化 agent。你的职责是将模糊的人类意图转化为机器可验证的合同。

规则：
- 目标必须可量化——"更好"不是目标，"覆盖率 ≥50%" 才是
- 边界必须明确——列出 NOT in scope
- 成功标准必须可程序化验证——CriticAgent 能明确判断 True/False
- 目标必须 99% 在当前约束下可实现——不是 stretch goal，是交付承诺
- 如果人类意图过于模糊或不可行，回复具体澄清问题而非编造目标
```

#### 3.5.4 PlannerAgent (KR) 设计约束

PlannerAgent 产出的 KR 必须满足 5 条约束：

| # | 约束 | 说明 |
|---|------|------|
| 1 | **原子化** | 一个 KR = 一个可执行动作，不可再拆 |
| 2 | **独立可验证** | CriticAgent 可单独判断此 KR 通过/失败 |
| 3 | **依赖显式** | 用 task_id 声明 DAG 依赖，不靠隐式假设 |
| 4 | **100% 完成即 O 达成** | KR 全通过 = Objective 达成。不设"超额 KR" |
| 5 | **分层递进** | 基础层（可运行）→ 覆盖层（覆盖率）→ 质量层（回归），不跳层 |

#### 3.5.5 KR 三层递进模型

每个 Objective 拆解为三层 KR，必须按层递进，不可跳跃。KR 三层递进模型是 PlannerAgent 的任务拆解标准结构（详见 §3.8.4 主循环流程定义 seq 2，PlannerAgent 产出的 TaskDAG 包含三层 KR）。层次描述如下：

| 层级 | 名称 | 目标 | 验证方式 | 失败处理 |
|------|------|------|---------|---------|
| 第一层 | **基础层** 🔴 | 证明测试 infrastructure 可用 | 1 个最小可行测试能跑通 | 终止 loop，上报人类 |
| 第二层 | **覆盖层** 🟡 | 逐文件提升覆盖率 | pytest + coverage 测量 | 重试 → 仍失败则跳过，记录 Memory |
| 第三层 | **质量层** 🟢 | 证明没有破坏已有功能 | 全量测试套件 100% 通过 | 回滚本轮新增测试，记录根因 |

**示例：** 基础层 KR 包括查询 KS 获取 ctx 结构、提取测试范例、生成 1 个 HTTP 测试并确认可运行；覆盖层逐文件设定目标如 `ontology_handler.py 覆盖 ≥50%`；质量层验证全量测试通过、新测试无 flaky、系统资源无泄漏。

**跳层的后果：** debugloop 的失败就是跳层的典型案例——直奔覆盖层（KR2），基础层（KR1 "测试能跑起来吗"）从未被验证。3 个测试全部在 setup 崩溃，覆盖率纹丝不动。如果当时有 KR 三层模型，第 0 轮就会暴露 "LLM 不知道 ctx 怎么构造" 这一致命缺陷，而非等到 500 元预算烧光。

#### 3.5.6 PlannerAgent 的 System Prompt 核心指令

```
你是任务规划 agent。你的职责是将 Objective 拆解为三层 KR。

规则：
- 必须按 KR 三层模型拆解：基础层 → 覆盖层 → 质量层，不可跳跃
- 每个 KR 必须原子化、独立可验证、100% 可实现
- KR 之间的依赖必须显式声明（DAG）
- 如果有历史失败记录，设计 KR 避开同样的路径
- 每个 KR 都是交付承诺，不是"试试看"
- 如果 Objective 在当前项目约束下无法 99% 实现，报告不可行并给出修正建议
- 最多 10 个 KR。如果不够，说明 Objective 需要缩小范围
```

#### 3.5.7 复盘驱动：KR 粒度自适应

Loop 持续运行时，KR 粒度会自动校准：

| 信号 | 诊断 | 修正 |
|------|------|------|
| 连续 3 轮某层 KR 全未达成 | KR 拆解粒度过大或项目约束未知 | PlannerAgent 将 KR 再拆一档，或查询 KS 补充未知信息 |
| 连续 3 轮某层 KR 全超预期达成 | PlannerAgent 低估了 Executor 能力 | PlannerAgent 提高 KR 粒度（合并小 KR 或提高单 KR 目标值） |
| 某 KR 反复失败但 CriticAgent 给通过 | KR 标准模糊 | PlannerAgent 重新定义更精确的验证条件 |
| 某 KR 反复通过但覆盖率不涨 | KR 方向错误（覆盖了不对的代码） | PlannerAgent 结合 coverage 报告调整目标文件 |

#### 3.5.8 GoalAgent 与 PlannerAgent 的 Prompt 约束

##### GoalAgent System Prompt（不可妥协）

```
你是目标标准化agent。将人类意图转为结构化StandardGoal。

硬约束：
- 目标必须可量化 — "更好"拒绝，"覆盖率≥50%"接受
- 边界必须明确 — 列出NOT in scope
- 成功标准必须可程序化验证 — CriticAgent能判断True/False
- 必须是99%在当前约束下可实现的交付承诺 — 不是stretch goal
- 意图模糊或不可行时 — 回复具体澄清问题，不编造目标
- 只定义WHAT不定义HOW — 不涉及实现路径

输出格式（必须严格遵守，不输出任何其他文本）：
{
  "goal": "一句话目标描述",
  "constraints": ["约束1", "约束2"],
  "success_criteria": ["可验证条件1", "可验证条件2"],
  "priority": "high" | "medium" | "low"
}

示例：
输入: "把pyOV handler层覆盖率搞到50%"
输出: {"goal":"将handler层分支覆盖率提升至≥50%","constraints":["不修改源码","不引入新依赖","预算≤¥20"],"success_criteria":["所有handler文件覆盖率≥50%","新测试可运行率100%","已有测试全通过"],"priority":"high"}
```

##### PlannerAgent System Prompt（不可妥协）

```
你是任务规划agent。将Objective拆解为三层KR。

硬约束：
- 必须按三层拆解：K1基础层(测试可运行)→K2覆盖层(覆盖率)→K3质量层(回归安全)，不可跳层
- 每个KR原子化 — 单一可执行动作
- 每个KR独立可验证 — CriticAgent可单独判断通过/失败
- 每个KR 100%可实现 — 交付承诺，不是"试试看"
- 依赖显式声明 — DAG，不靠隐式假设
- 最多10个KR — 超出则缩小Objective范围
- 历史失败记录 → 设计KR避开同样路径
- 先查询Knowledge Server → 获取项目上下文后再拆解
- 先查询Memory → 获取上轮失败信息和经验教训
- 不可行时 → 报告原因并给出修正建议，不编造KR
- 每个KR必须定义verification字段：method(验证方式)、criteria(逐条判定)、evidence(证据来源)
- verification.method 只能为以下值之一：pytest / coverage / lint / static_analysis
- verification.criteria 每项为一条可判断True/False的语句(粒度适中，不超一行)
- 最少1个KR，最多10个KR

三层模板：
K1基础层: 证明测试infrastructure可用（1个最小可行测试能跑通）。失败→终止loop
K2覆盖层: 逐文件提升覆盖率。单KR失败→重试→仍失败→跳过，记录Memory
K3质量层: 证明未破坏已有功能。全量测试100%通过

输出格式（必须严格遵守，不输出任何其他文本）：
{
  "tasks": [
    {
      "task_id": "k1_1",
      "description": "查询KS获取handler层ctx结构",
      "priority": 1,
      "dependencies": [],
      "max_retries": 2,
      "verification": {
        "method": "static_analysis",
        "criteria": ["ctx结构文档包含所有key的名称和初始化代码"],
        "evidence": "rdf/ast/runtime-ctx.ttl"
      }
    }
  ]
}
```

##### KR 拆解输出校验

PlannerAgent 产出 `TaskDAG` 后，LoopController 在进入 seq 3 前执行以下校验：

```
check_1: 是否有 K1 层 KR（基础层）？ → 无则拒绝，退回PlannerAgent
check_2: KR 数量 ≤ 10？ → 超出则拒绝，退回PlannerAgent
check_3: 每个 KR 的 dependencies 中的 task_id 是否都存在于 TaskDAG 中？ → 否则拒绝
check_4: 是否有循环依赖？ → 拓扑排序检测，有则拒绝
check_5: 每个 KR 的 description 和 verification 字段非空？ → 否则拒绝
```

### 3.6 三视角对抗验证的角色边界与独立性原则

#### 3.6.1 核心原则：审计者永不执行

**auditor 的独立性是不可妥协的红线。** auditor 如果执行代码，就无法客观评价代码——自己跑出来的结果自己评，怎么区分"我执行得对不对"和"代码写得对不对"？

三个 Critic 视角的分工必须以**验证手段**而非**问题类型**来划分：

| 视角 | Agent | 验证手段 | 核心动作 | 关键约束 |
|------|-------|---------|---------|---------|
| **correctness** 正确性 | reviewer | 静态分析 | 阅读代码 diff，对比 task spec，找逻辑错误、API 误用、设计缺陷 | **不运行代码** |
| **completeness** 完整性 | tester | 动态执行 | 运行测试，测量覆盖率，在隔离环境中验证可运行性和正确性 | **必须运行代码，负责环境隔离** |
| **robustness** 健壮性 | auditor | 审计审查 | 逐项过 checklist，审查安全、错误处理、流程合规、设计一致性 | **永不执行代码，保持独立判断** |

#### 3.6.2 为什么环境隔离属于 tester 而不是 auditor

环境隔离是**执行层面的问题**：代码在什么环境下跑、依赖是否完整、多项目间是否冲突。这些问题必须通过实际运行来验证，属于 tester 的动态执行范畴。

auditor 的工作是**审查 tester 的隔离策略是否完备**——检查 tester 是否在 checklist 中覆盖了隔离项、是否在多环境中验证了、是否有遗漏的隔离维度。但 auditor 自己不去搭建和运行隔离环境。

```
tester 的职责：
  └─ 在干净环境中运行代码
  └─ 验证环境隔离是否有效（多项目不冲突、依赖完整）
  └─ 产出环境隔离报告

auditor 的职责：
  └─ 审查 tester 的隔离报告是否完备
  └─ checklist 中检查隔离维度是否全覆盖
  └─ 发现 tester 遗漏的隔离项 → 反馈给 tester 补充
  └─ 不自己跑隔离环境（保持独立判断）
```

#### 3.6.3 auditor 独立性的三条红线

| # | 红线 | 原因 |
|---|------|------|
| 1 | **不执行代码** | 执行=参与=失去客观立场。auditor 一旦自己跑代码，就无法评价"代码跑对了没有"，因为它同时在评价"自己跑得对不对"。 |
| 2 | **不调用 exec/process 工具** | 审计只需 `read` 读取代码、报告、checklist。任何需要运行的操作都是越界。 |
| 3 | **不修改任何文件** | 审计发现问题 → 反馈给 main → main 委派 coder/tester 修复。auditor 只发现问题，不解决问题。 |

#### 3.6.4 对比：Loop Engineering 原始设计的隐含问题

Loop Engineering 源码中 CriticAgent 的三个 lens（correctness/completeness/robustness）定义存在重叠，且未明确区分验证手段：

```
源码定义（存在问题）：
  correctness:  "事实错误、逻辑谬误、错误假设"
  completeness: "缺失边界条件、未处理场景、遗漏需求"
  robustness:   "隐藏依赖、脆弱假设、下游可用性"

问题：
  - "错误假设"（correctness）就是 "脆弱假设"（robustness）
  - "未处理场景"（completeness）就会导致 "事实错误"（correctness）
  - 三者不是按"验证手段"分，而是按"问题类型"分 → 天然重叠
```

**team-loop 的改进：按验证手段划分，而非按问题类型划分。**

```
team-loop 定义（以手段定边界）：
  correctness  → reviewer  → 静态读代码，不运行
  completeness → tester    → 动态执行，环境隔离，跑覆盖率
  robustness   → auditor   → 审计审查，checklist，不执行
```

手段不同 → 工具链不同 → 三个 agent 天然不可互换 → 边界清晰。

#### 3.6.5 验证流程中的 auditor 位置

详见 §3.8.4 seq 4-6 的三视角并行验证流程（seq 4 = 三视角并行验证，seq 5 = 2/3裁决，seq 6 = 终止判断）。

**auditor 审的是"审查者有没有审查到位"，不是"代码跑得对不对"。** 这保证了 auditor 永远站在最外层，保持独立判断。

#### 3.6.6 三视角审查清单

审查、验证、审计——三个 Critic 视角的术语统一如下：

- reviewer → **"正确性审阅"**（不叫"审查"）
- tester → **"完整性验证"**
- auditor → **"健壮性审计"**（auditor 叫"审计者"，不叫"审查者"。"审查"留给 reviewer）

##### reviewer（correctness — 正确性审阅，不运行代码）

审查项：
- 代码与 task spec 的一致性 [task要求的功能是否全部实现]
- 逻辑错误：条件判断、循环边界、空值处理
- API 误用：参数类型、返回值处理、异常捕获
- 设计缺陷：循环依赖、过度耦合、职责不清
- import 正确性：是否引入了项目不存在的模块 [幻觉检测]
- 与已有代码的兼容性：是否破坏已有接口签名
- 不检查：覆盖率、测试通过率、性能 [那是tester和auditor的事]

##### tester（completeness — 完整性验证，动态执行，环境隔离）

审查项：
- 测试可运行性：pytest 能否成功收集并执行 [基础层关键指标]
- 覆盖率变化：target 文件的分支覆盖率变化量
- 断言有效性：测试是否真正验证了预期行为而非只调用函数
- 环境隔离：新测试在干净环境下能否独立运行
- 回归安全：已有测试是否全部通过 [全量套件]
- flaky检测：同一测试连续3次运行结果一致
- 不检查：代码逻辑是否正确 [那是reviewer的事]、安全性 [那是auditor的事]

##### auditor（robustness — 健壮性审计，永不执行代码）

checklist 逐项检查（12项）：
- [ ] 错误处理完整性：异常分支是否被覆盖
- [ ] 资源管理：文件句柄、socket连接是否正确关闭
- [ ] 注入安全：是否存在未校验的外部输入
- [ ] reviewer审阅完整性：reviewer是否遗漏了关键逻辑路径 [审计审查者]
- [ ] tester验证完整性：tester是否在真实隔离环境中运行 [审计审查者]
- [ ] 交接物Schema合规：所有产出是否符合§3.7.4定义的Schema
- [ ] CR记录完整性：变更是否记录了CR [审计变更管理]
- [ ] 日志可追溯性：交互链是否可按interaction_id串联
- [ ] 流程合规：是否按§3.8.4定义的流程执行（无跳步）
- [ ] 模型使用合规：critic是否使用了与coder不同的模型
- [ ] 不执行代码 — 本条为auditor自身合规检查
- [ ] 不修改文件 — 本条为auditor自身合规检查

### 3.7 工程质量基础设施

team-loop 作为无人值守的自主循环系统，工程质量基础设施不是"锦上添花"而是"生存必需"。没有日志无法复盘、没有 DI 无法测试、没有隔离无法独立验证、没有标准交接物 agent 之间无法通信、没有变更管理无法追溯。以下五项基础设施是构建系统的硬前置条件。

---

**非功能性需求的前置设计原则：** 以下五项基础设施不是"实现时再考虑"的——它们必须在系统设计阶段精细规划。每项基础设施遵循同一条核心原则：**关注点分离**。信息的采集与加工分离、数据的生产与消费分离、框架的机制与策略分离。采集端尽量少侵入业务代码——agent 只管调用 `logger.log()`，不感知输出格式、不感知当前模式、不感知谁来消费。加工端提供专门的日志分析模块，不同角色（loop/人/运维）的"怎么看"是加工层的问题，不是采集层的问题。这一原则贯穿所有基础设施项。

#### 3.7.1 分级完备的日志框架

**设计哲学：采集与加工分离。**

agent 的职责是**完整采集**——在关键节点调用 `logger.info()`/`logger.error()` 等，不判断"这条日志要不要写"、不判断"谁要这条日志"。日志框架的职责是**可靠传输**——采集到的每一条日志绝不丢失。日志分析模块的职责是**多视角加工**——不同角色（loop 自身、开发者、运维）各取所需，按交互编号筛选、按级别过滤、按时序聚合。

```
agent 代码                   日志框架                     日志分析模块
─────────                   ─────────                   ────────────
只管写：                     只管传：                     只管看：
logger.info(...)            采集 → 缓冲 → 落盘            ├── loop 视角：JSON 事件流
logger.error(...)           保证不丢、不阻塞 agent         ├── 人 视角：彩色终端 + 可读日志
                            不判断"要不要写"              ├── 运维视角：时序指标 + 告警
                                                        └── 交互筛选：按 interaction_id 串联
```

##### 3.7.1.1 交互唯一编号（🔴 优先设计）

**为什么优先：** 一次 `/goal` 命令触发一个 loop，一个 loop 包含多轮，每轮包含多个 task。如果日志没有全局唯一的交互编号，事后无法回答"第 5 轮 t3 失败是因为第 2 轮 t1 的什么决定？"——日志只是散落的事件，无法串联成决策链。交互编号是所有日志基础设施中最优先设计的项，因为它决定了后续所有分析能力的上限。

**交互编号体系：**

```
interaction_id                          ← 一次交互的唯一标识
  ├── source: "manual" | "cron" | "test"   ← 交互来源
  ├── source_detail:                         ← 来源详情
  │     manual: "iFeel /goal 提高pyOV覆盖率"
  │     cron:   "cron-job-nightly-coverage"
  │     test:   "test-suite-regression-001"
  ├── loop_id                               ← 该次交互触发的 loop
  │     ├── round: 1
  │     │     ├── task_id: t1
  │     │     ├── task_id: t2
  │     │     └── ...
  │     ├── round: 2
  │     └── ...
  └── status: "running" | "completed" | "failed" | "terminated"
```

**编号生成规则：**

```
interaction_id = {source}-{timestamp}-{hash4}
  示例: manual-20260628T110000-a3f2

loop_id = {interaction_id}-L
  示例: manual-20260628T110000-a3f2-L

task_id = {loop_id}-R{round}-{task_seq}
  示例: manual-20260628T110000-a3f2-L-R3-t5
```

**每条日志携带的追踪字段：**

```json
{
  "interaction_id": "manual-20260628T110000-a3f2",
  "loop_id": "manual-20260628T110000-a3f2-L",
  "round": 3,
  "task_id": "manual-20260628T110000-a3f2-L-R3-t5",
  "agent": "auditor",
  "level": "WARN",
  "event": "critic_fail",
  "data": {...}
}
```

**交互来源类型：**

| source | 触发方式 | interaction_id 前缀 | 示例 |
|--------|---------|-------------------|------|
| `manual` | 人类直接输入 `/goal` 命令 | `manual-` | `manual-20260628T110000-a3f2` |
| `cron` | 定时任务触发 | `cron-{job_name}-` | `cron-nightly-coverage-20260628T020000-b7e1` |
| `test` | 测试套件执行 | `test-{suite}-` | `test-regression-suite-001-20260628T150000-c9d4` |
| `event` | 外部事件触发（webhook/文件变更） | `event-{source}-` | `event-github-push-20260628T160000-d2f5` |

**筛选能力：** 日志分析模块支持按 `interaction_id` 精确筛选一次完整交互的所有日志、按 `source` 聚合所有同类型操作的统计、按 `loop_id + round` 定位特定轮次的决策细节。

##### 3.7.1.2 异常全量捕捉（不可逃脱）

**为什么关键：** 系统级中断（进程崩溃、OOM、socket 断开、LLM API 超时无响应）如果逃脱了日志框架，就是黑洞——"系统停了，但不知道为什么停"。自主循环无人值守，这种黑洞可能持续数小时才被发现。

**捕捉机制（分层防御）：**

```
第一层：Python 异常钩子
  sys.excepthook → 捕获所有未处理异常
  → 写入 FATAL 日志（含完整 traceback + 最后 20 条日志上下文）
  → 如果可能，刷新日志缓冲区到磁盘
  → 如果可能，发送告警

第二层：信号处理器
  signal(SIGTERM, handler) → 记录"进程收到 SIGTERM"
  signal(SIGINT, handler)  → 记录"进程收到 SIGINT（Ctrl+C）"
  → 写 FATAL 日志 → 优雅关闭

第三层：atexit 兜底
  atexit.register() → 进程退出前最后一次机会
  → 检查是否有未落盘的日志缓冲区
  → 写入 "process_exit" 事件（含退出码）

第四层：外部看门狗
  独立 watchdog 进程/定时任务
  → 检测 team-loop 进程存活
  → 检测日志文件最后写入时间（超过 N 分钟无写入 → 可能卡死）
  → 记录 watchdog 告警（含最后一条日志的 interaction_id）
```

**不可逃脱的保证：**

| 场景 | 捕获方式 | 日志内容 |
|------|---------|---------|
| Python 异常（未 except 的） | `sys.excepthook` | FATAL + traceback + 最后 20 条日志上下文 |
| 进程被 kill -9 | atexit 不可靠，依赖第四层看门狗 | watchdog 检测到进程消失 + 最后日志时间 |
| OOM 被系统杀死 | 同 kill -9，看门狗检测 | watchdog 告警 + 内存用量趋势 |
| LLM API 超时无响应 | try/except + timeout | ERROR + 超时时间 + 重试次数 |
| Knowledge Server socket 断开 | try/except + 重连逻辑 | ERROR + 断开时间 + 重连尝试 |
| 磁盘满 | 日志写入失败时触发 | FATAL + 磁盘用量 + 暂停 loop |
| 死循环（agent 卡住） | 看门狗检测 + round 超时 | FATAL + 当前 round + 最后执行 task |

**异常日志的最低信息要求（MDR — Minimum Diagnostic Record）：**

```
任何 ERROR/FATAL 日志必须包含：
  1. interaction_id — 哪个交互
  2. agent — 哪个 agent 崩溃
  3. round + task_id — 在做什么时崩溃
  4. exception type + message — 什么异常
  5. traceback — 完整调用栈
  6. last_n_logs(20) — 崩溃前最后 20 条日志（上下文）
  7. system_state — 内存用量、磁盘用量、进程树（尽力而为）
```

##### 3.7.1.3 三重输出设计

**三个受众的日志需求不同：**

| 受众 | 关注点 | 格式需求 | 示例问题 |
|------|--------|---------|---------|
| **loop 自身** | 结构化事件流，用于 PlannerAgent 读取记忆、CriticAgent 判断重试 | JSON 行，字段完备 | "t5 在第3轮被auditor否决，原因是什么？" |
| **人（开发者）** | 可读、可追溯、可快速定位问题 | 彩色终端输出 + 文件 + 上下文展开 | "第3轮发生了什么？为什么停了？" |
| **运维（监控）** | 聚合指标、趋势、告警 | 时序数据 + 仪表盘 + 告警规则 | "过去24小时KR通过率趋势？哪个agent最慢？" |

**三重输出设计：**

```
┌─────────────────────────────────────────────┐
│              统一日志总线                     │
│    所有 agent 通过 Logger.log(level, event)   │
│    写入结构化事件，不关心输出格式              │
└──────────┬──────────────────────────────────┘
           │
     ┌─────┼─────────────┐
     ▼     ▼             ▼
┌─────────┐ ┌──────────┐ ┌──────────┐
│ JSON行  │ │ 人类可读  │ │ 运维指标  │
│ 输出     │ │ 输出      │ │ 输出      │
├─────────┤ ├──────────┤ ├──────────┤
│ 受众:   │ │ 受众:     │ │ 受众:    │
│ loop    │ │ 开发者    │ │ 运维     │
├─────────┤ ├──────────┤ ├──────────┤
│ 格式:   │ │ 格式:     │ │ 格式:    │
│ .jsonl  │ │ 终端彩色  │ │ 时序DB   │
│ 每行独立 │ │ + .log    │ │ + 告警   │
│ 可grep  │ │ 文件      │ │ 规则     │
└─────────┘ └──────────┘ └──────────┘
```

**输出通道一：JSON 行（给 loop）**

loop 自身的结构化事件流，每条日志独立、可 grep、可回放：

```json
{"ts":"2026-06-28T11:00:00Z","level":"INFO","agent":"main","loop_id":"L001","round":3,"task_id":"t5","event":"task_verdict","data":{"kr":"ontology_handler.py覆盖≥50%","passed":2,"total":3}}
{"ts":"2026-06-28T11:00:05Z","level":"WARN","agent":"auditor","loop_id":"L001","round":3,"task_id":"t5","event":"critic_fail","data":{"lens":"robustness","reason":"错误处理路径未覆盖异常分支"}}
{"ts":"2026-06-28T11:00:10Z","level":"ERROR","agent":"main","loop_id":"L001","round":3,"task_id":"t5","event":"kr_failed","data":{"reason":"2/3未通过","retry":1,"max_retries":2}}
```

**输出通道二：人类可读（给开发者）**

终端彩色输出 + 文件落盘，不同级别不同颜色，带上下文展开：

```
[2026-06-28 11:00:00] [INFO ] [main    ] L001 R3 T5 │ KR 裁决：2/3 通过
[2026-06-28 11:00:05] [WARN ] [auditor ] L001 R3 T5 │ ❌ robustness 否决
                                                     │    原因：错误处理路径未覆盖异常分支
                                                     │    建议：补充 handle_tree ctx 缺失 key 场景
[2026-06-28 11:00:10] [ERROR] [main    ] L001 R3 T5 │ KR 失败（2/3未通过），重试 1/2
[2026-06-28 11:00:15] [FATAL] [main    ] L001 R3 -- │ 连续 3 轮基础层 KR 未达成，终止 loop
                                                     │    详情：K1.1 Knowledge Server 连接超时
                                                     │    建议：检查 KS 进程状态，手动重启后重新 /goal
```

人类日志的格式原则：
- **一行概要** — 不用展开就知道发生了什么
- **缩进详情** — 关键上下文在下一行缩进展开，不淹没主流程
- **级别着色** — ERROR 红色、WARN 黄色、INFO 白色、DEBUG 灰色，扫一眼就能定位
- **决策透明** — 不只是"失败了"，而是"谁、为什么、什么建议"

**输出通道三：运维指标（给监控）**

不记录单条事件，而是聚合时序指标，对接监控系统：

| 指标 | 粒度 | 告警条件 | 用途 |
|------|------|---------|------|
| `loop.round.duration_sec` | 每轮 | 单轮超过 10 分钟 | 检测 agent 卡死或死循环 |
| `loop.kr.pass_rate` | 每轮 | 连续 3 轮低于 30% | 检测目标不可行或 KR 拆解错误 |
| `loop.critic.vote_distribution` | 每轮 | 某视角连续 5 次独投反对 | 检测 Critic 偏见或过度严格 |
| `loop.token.consumed` | 每轮 | 超过预算 80% | 成本预警 |
| `agent.response_time_p95` | 滚动窗口 | 超过 60 秒 | 检测 LLM API 延迟 |
| `loop.basic_layer_failures` | 累计 | 基础层 KR 失败 ≥ 1 | 立即告警——基础设施问题 |

##### 3.7.1.4 日志级别语义（三方共识）

| 级别 | 对 loop | 对人 | 对运维 |
|------|---------|------|--------|
| **DEBUG** | 回放用的事件流水 | 不需要看（除非追 bug） | 不采集 |
| **INFO** | PlannerAgent 读取记忆 | 日常扫一眼知道进度 | 采集为指标源 |
| **WARN** | 触发 KR 重试逻辑 | 黄色高亮，需要关注 | 累计 3 次以上告警 |
| **ERROR** | 触发任务跳过或重规划 | 红色高亮，必须看 | 立即告警 |
| **FATAL** | 终止 loop | 红色闪烁，立即处理 | P0 告警 |

**日志文件组织：**

```
logs/
├── L001/                          ← 一个 loop 一个目录
│   ├── R001/                       ← 每轮独立子目录
│   │   ├── main.jsonl
│   │   ├── main.log               ← 人类可读版
│   │   ├── coder.jsonl
│   │   ├── coder.log
│   │   ├── reviewer.jsonl
│   │   ├── reviewer.log
│   │   ├── tester.jsonl
│   │   ├── tester.log
│   │   ├── auditor.jsonl
│   │   ├── auditor.log
│   │   ├── publicist.jsonl
│   │   ├── publicist.log
│   │   ├── metrics.json            ← 运维指标快照
│   │   └── round-summary.md        ← 人读摘要
│   ├── R002/
│   │   └── ...
│   └── loop-summary.md             ← 全 loop 人读总结
├── metrics/                        ← 时序指标（可对接 Prometheus）
│   └── team-loop-metrics.jsonl
└── alerts/                         ← 告警历史
    └── alerts.jsonl
```

##### 3.7.1.5 日志模式控制（内聚在日志框架内）

日志框架提供两种运行模式，切换逻辑完全内聚在 `Logger` 模块中，agent 代码不感知当前模式：

```python
# agent 代码只管写，不管开不开
logger.info("KR 裁决", data={"kr": "t5", "passed": 2, "total": 3})
logger.debug("Critic 投票明细", data={"verdicts": [...]})

# 是否真正输出、以什么级别输出——由 Logger 模块集中决定
```

| 模式 | 触发条件 | 日志级别 | 三重输出状态 | 用途 |
|------|---------|---------|------------|------|
| **looplog（默认）** | loop 启动，交付前 | DEBUG | JSON ✅ + 人类 ✅ + 指标 ✅ | 开发调试、问题定位、复盘分析 |
| **lean（精简）** | 交付后人工切换 | INFO（WARN+ 人类展开） | JSON ✅ + 人类 ⚠️ + 指标 ✅ | 日常运行，减少日志量 |

**模式切换规则：**

```
loop 启动 → 自动进入 looplog 模式（默认）
     │
     │  交付判定：连续 N 轮无 ERROR，KR 通过率稳定 > 80%
     │
     ▼
人工确认 → 切换为 lean 模式
     │
     │  异常触发：连续 3 次 WARN 或 1 次 ERROR
     │
     ▼
自动回切 → looplog 模式（无需人工干预）
```

**内聚约束（不可妥协）：**

| # | 约束 | 原因 |
|---|------|------|
| 1 | **只有 Logger 模块包含模式判断逻辑** | agent 代码中出现 `if mode == "looplog"` 即违规 |
| 2 | **agent 永远调用同一套 API**（`logger.debug/info/warn/error/fatal`） | agent 不感知当前级别阈值 |
| 3 | **模式切换对 agent 透明** | agent 不因模式切换而改变行为 |
| 4 | **Lean 模式保留 ERROR 级别的人类展开** | 出问题时人类仍需完整上下文，不能只剩一行概要 |
| 5 | **Lean 模式保留全部 JSON 输出** | loop 的 MemoryAgent 仍需要结构化事件做检索 |

#### 3.7.2 可注入、可调试、可测试、可 Mock 的代码组织框架

**关注点分离：** agent 只声明依赖（构造函数参数），不创建依赖。DI 容器负责装配——用什么实现、在什么环境下装配，是容器的事，不是 agent 的事。测试时换 mock、生产时换真实实现——agent 代码不变。

**核心接口契约（Interface Contracts）：**

系统定义 6 个核心接口，所有组件依赖接口不依赖实现。接口定义即测试边界——实现一个接口，就是实现一组可独立验证的方法签名。

```python
# === 核心接口定义 ===

class LLMProvider(ABC):
    """LLM 调用抽象 — 核心逻辑不感知具体模型"""
    @abstractmethod
    def chat(self, messages: list[Message], system: str) -> LLMResponse: ...
    @abstractmethod
    def model_name(self) -> str: ...
    # 生产实现: DeepSeekProvider, GLMProvider
    # 测试实现: MockLLMProvider, ReplayLLMProvider

class KnowledgeProvider(ABC):
    """知识查询抽象 — 核心逻辑不感知存储方式"""
    @abstractmethod
    def query(self, module: str, keywords: list[str]) -> KnowledgeContext: ...
    @abstractmethod
    def index(self, project_path: str) -> None: ...
    # 生产实现: RDFServerAdapter
    # 测试实现: MockKnowledgeProvider

class AgentAdapter(ABC):
    """Agent 通信抽象 — 核心逻辑不感知编排框架"""
    @abstractmethod
    def dispatch(self, agent_id: str, task: TaskUnit) -> AgentResult: ...
    @abstractmethod
    def await_result(self, task_id: str, timeout_sec: int) -> TaskResult: ...
    # 生产实现: OpenClawAdapter (sessions_spawn)
    # 测试实现: DirectCallAdapter (同步函数调用)

class StoreProvider(ABC):
    """持久化抽象 — 核心逻辑不感知存储后端"""
    @abstractmethod
    def get(self, collection: str, key: str) -> dict | None: ...
    @abstractmethod
    def set(self, collection: str, key: str, value: dict) -> None: ...
    @abstractmethod
    def list(self, collection: str, filter_fn=None) -> list[dict]: ...
    # 生产实现: JsonFileStore, SqliteStore
    # 测试实现: MemoryStore

class LoggerProvider(ABC):
    """日志抽象 — agent 不感知输出格式"""
    @abstractmethod
    def log(self, level: str, agent: str, event: str,
            interaction_id: str, loop_id: str, round: int,
            task_id: str | None, data: dict) -> None: ...
    # 生产实现: JsonLineLogger + HumanReadableLogger + MetricsLogger
    # 测试实现: MemoryLogger (验证日志调用但不输出)

class EventBus(ABC):
    """事件总线 — 组件间解耦通信"""
    @abstractmethod
    def publish(self, event: str, data: dict) -> None: ...
    @abstractmethod
    def subscribe(self, event: str, handler: Callable) -> None: ...
```

**DI 容器装配契约：**

```python
# 容器配置（JSON/YAML），声明式装配，不写代码
{
  "providers": {
    "llm": "deepseek.DeepSeekProvider",       # 生产
    "llm": "mock.MockLLMProvider",            # 测试（覆盖）
    "knowledge": "rdf.RDFServerAdapter",      # 生产
    "agent": "openclaw.OpenClawAdapter",      # 生产
    "store": "json.JsonFileStore",
    "logger": "jsonline.JsonLineLogger",
    "eventbus": "memory.MemoryEventBus"
  },
  "mode": "looplog",                          # lean | looplog
  "budget": {"max_tokens": 100000, "max_rounds": 20}
}

# LoopController 构造 — 容器注入全部依赖
controller = LoopController(
    llm=container.resolve(LLMProvider),
    knowledge=container.resolve(KnowledgeProvider),
    agent=container.resolve(AgentAdapter),
    store=container.resolve(StoreProvider),
    logger=container.resolve(LoggerProvider),
    eventbus=container.resolve(EventBus),
    config=container.config
)
```

**可测试性契约：**

每个组件必须同时提供：生产实现 + mock 实现 + 单元测试套件。测试不启动完整系统，只验证组件对外接口的行为：

| 组件 | 单元测试验证 | 不依赖 |
|------|------------|--------|
| GoalAgent | Mock LLMProvider → 输入文本 → 验证 StandardGoal 结构 | 不启动 loop |
| PlannerAgent | Mock LLM + Memory + KS → 验证 Task[] 结构 + DAG | 不启动 agent |
| LoopController | Mock 全部 6 个接口 → 验证调度顺序和终止条件 | 不调用真实 LLM |
| Critic (3视角) | Mock LLMProvider → 验证各视角 prompt 不含对方职责 | 不运行代码 |
| KnowledgeServer | Mock socket → 验证查询-响应契约 | 不启动 rdflib |
| AgentAdapter | DirectCallAdapter → 同步调用 → 验证 dispatch-await 契约 | 不启动 OpenClaw |

**代码覆盖插装标准：**

```python
@traceable("agent.run")          # 入口埋点
def run(self, input, ctx): ...

@traceable("critic.vote")        # 投票埋点
def evaluate_lens(self, ...): ...

@traceable("adapter.call")       # 外部调用埋点，自动计次计时
def chat(self, messages, system): ...
```

所有 `@traceable` 装饰器记录：调用时间、参数哈希、返回值状态、耗时。不依赖外部工具，纯 Python 装饰器实现。

---

#### 3.7.3 稳定的上下游戏离与设计模式锚点

**关注点分离：** 核心逻辑层只定义接口，适配器层负责对接具体技术。核心逻辑不 import 任何框架 SDK、数据库驱动、网络库。换框架、换数据库、换 OS——核心逻辑代码不变。

**端口定义（明确的 import 白名单与黑名单）：**

```
核心逻辑层 — 允许 import:
  ✅ Python stdlib: abc, dataclasses, typing, enum, json, logging (仅 LoggerProvider)
  ✅ 项目内部: core/types.py, core/interfaces.py, core/events.py
  ✅ 第三方纯数据: pydantic (仅 Schema 定义)

核心逻辑层 — 禁止 import:
  ❌ 框架 SDK: openclaw, anthropic, openai
  ❌ 数据库: rdflib, sqlite3, sqlalchemy
  ❌ 网络: socket, requests, aiohttp
  ❌ 操作系统: os, sys, subprocess (仅 atexit/signal 等异常处理除外)
  ❌ 文件系统: pathlib (仅类型定义除外)
```

**端口/适配器边界检查规则（CI 强制执行）：**

```bash
# 检查核心逻辑层是否引入了禁止的依赖
$ python -c "
import ast, sys
forbidden = {'openclaw', 'anthropic', 'openai', 'rdflib', 'sqlite3',
             'sqlalchemy', 'socket', 'requests', 'aiohttp'}
with open(sys.argv[1]) as f:
    tree = ast.parse(f.read())
for node in ast.walk(tree):
    if isinstance(node, ast.Import):
        for alias in node.names:
            if alias.name.split('.')[0] in forbidden:
                print(f'VIOLATION: {sys.argv[1]} imports {alias.name}')
                sys.exit(1)
"
```

**适配器实现契约：**

每个适配器必须实现对应的核心接口，且只做"翻译"不做"判断"：

```
┌──────────────────────────────────────┐
│  核心接口 (core/interfaces.py)        │
│  定义: AgentAdapter.dispatch()        │
│  定义: KnowledgeProvider.query()      │
│  定义: LLMProvider.chat()            │
│  ...                                 │
│  → 只定义"做什么"，不定义"怎么做"     │
└──────────────┬───────────────────────┘
               │ implements
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│OpenClaw│ │Direct  │ │RDF     │
│Adapter │ │Call    │ │Adapter │
│        │ │Adapter │ │        │
├────────┤ ├────────┤ ├────────┤
│翻译:   │ │翻译:   │ │翻译:   │
│session │ │函数调用│ │SPARQL  │
│_spawn  │ │→ 返回  │ │→ dict  │
│→ result│ │        │ │        │
└────────┘ └────────┘ └────────┘
```

**设计模式锚点（含使用约束）：**

| 模式 | 使用位置 | 不可用于 | 示例 |
|------|---------|---------|------|
| **Adapter** | `adapters/` 目录下 | 核心逻辑层 | `adapters/openclaw.py` 实现 `AgentAdapter` |
| **Strategy** | 核心逻辑层的决策点 | 适配器层 | `loopctrl.termination_strategies` 列表，运行时选择 |
| **Observer** | EventBus 订阅者 | 同步调用链 | `eventbus.subscribe("kr.completed", log_handler)` |
| **Command** | Task 执行单元 | 直接调用 | 每个 KR 封装为 `Command(task, context)`，支持 undo |
| **Repository** | StoreProvider 实现 | 业务逻辑 | `store.get("tasks_3", "t5")` — 不关心是文件还是 SQLite |

---

#### 3.7.4 统一的成体系的标准角色交接物

**关注点分离：** 交接物只定义数据契约（Schema），不定义消费方式。producer 按 Schema 生产，consumer 按 Schema 校验——producer 不假设 consumer 是谁，consumer 不假设 producer 是谁。

**7 种交接物 Schema 定义（Pydantic，不可妥协的契约）：**

```python
from pydantic import BaseModel, Field
from datetime import datetime
from enum import Enum

# ─── H1: Goal ───
class StandardGoal(BaseModel):
    schema_version: str = "1.0"
    producer_agent: str = "main"
    interaction_id: str
    timestamp: datetime
    goal: str
    constraints: list[str]
    success_criteria: list[str]
    priority: Literal["high", "medium", "low"]

# ─── H2: Task[] ───
class Task(BaseModel):
    schema_version: str = "1.0"
    task_id: str
    description: str
    priority: int
    dependencies: list[str]
    status: str
    max_retries: int = 2     # 此KR失败后的最大重试次数
    verification: dict | None = None

class TaskDAG(BaseModel):
    schema_version: str = "1.0"
    producer_agent: str = "main"
    interaction_id: str
    round: int
    timestamp: datetime
    tasks: list[Task]
    # Task: {task_id, description, priority, dependencies, status, max_retries}

# ─── H3: TaskUnit ───
class TaskUnit(BaseModel):
    """LoopCtrl → Executor 的下发指令"""
    schema_version: str = "1.0"
    producer_agent: str = "main"
    interaction_id: str
    round: int
    timestamp: datetime
    task: Task
    context: TaskContext
    # TaskContext: {goal, knowledge, past_failures, constraints}

**Task.verification 字段约束（Planner→Executor→Critic 三边契约）：**

PlannerAgent 必须为每个 KR 定义 `verification` 字段，描述此 KR "怎么算完成"。Executor 执行后对照 verification 自检，Critic 对照 verification 验证。

```json
// Task 的完整结构（含 verification）
{
  "task_id": "t5",
  "description": "为 ontology_handler.py 的 handle_tree 正常路径编写 HTTP 测试",
  "priority": 1,
  "dependencies": ["t1", "t2"],
  "max_retries": 2,
  "verification": {
    "method": "pytest + coverage",
    "criteria": [
      "测试文件 tests/test_handlers.py 中新增至少 1 个 test_handle_tree_* 函数",
      "pytest 收集并执行成功，无 import/setup 错误",
      "ontology_handler.py 的分支覆盖率增量 ≥ +1%"
    ],
    "evidence": "coverage.json 报告中 ontology_handler.py 的 covered_branches 增加"
  }
}
```

**verification 字段规则：**
- `method`: 验证方式（pytest/coverage/lint/static_analysis），对应 Critic 的验证手段
- `criteria`: 可逐条判断 True/False 的完成条件列表
- `evidence`: 可机器读取的证据来源（文件路径/报告字段）
- 无 verification 的 KR → LoopController 拒绝下发

# ─── H4: Delivery ───
class TaskResult(BaseModel):
    """Executor → Critics 的交付物"""
    schema_version: str = "1.0"
    producer_agent: str  # "coder"
    interaction_id: str
    round: int
    task_id: str
    timestamp: datetime
    output: str          # 代码 diff、文件路径、执行日志
    files_changed: list[str]
    status: Literal["success", "failed"]
    metadata: TaskMetadata
    # TaskMetadata: {tokens_used, duration_ms, model}
    reasoning: str          # Executor 自述

| `reasoning` | ✅ | string | Executor 的自述：做了什么、为什么这样做、遇到什么问题。供 Critic 理解执行意图，供 Planner 学习失败原因 |

# ─── H4.1: CoverageReport ───
class CoverageReport(BaseModel):
    """tester → LoopController + auditor 的覆盖率报告"""
    schema_version: str = "1.0"
    producer_agent: str = "tester"
    interaction_id: str
    round: int
    task_id: str
    timestamp: datetime
    files_measured: list[str]       # 测量了哪些文件
    baseline_coverage: float         # 基线分支覆盖率%
    new_coverage: float              # 本轮后分支覆盖率%
    delta: float                     # 变化量
    uncovered_branches: int          # 剩余未覆盖分支数

# ─── H5: Verdict ───
class CriticVerdict(BaseModel):
    """单个 Critic → LoopCtrl 的投票"""
    schema_version: str = "1.0"
    producer_agent: str  # "reviewer" | "tester" | "auditor"
    interaction_id: str
    round: int
    task_id: str
    timestamp: datetime
    lens: Literal["correctness", "completeness", "robustness"]
    passed: bool
    reasoning: str
    issues: list[str]
    improvements: list[str]

# ─── H6: Memory ───
class MemoryEntry(BaseModel):
    """Publicist → Planner 的记忆条目"""
    schema_version: str = "1.0"
    producer_agent: str = "publicist"
    interaction_id: str
    timestamp: datetime
    task_id: str
    task_summary: str
    outcome: str           # 结果：通过 / 未通过 — 及原因
    lessons: list[str]
    keywords: list[str]   # 用于语义检索

持久化格式：rdf/ast/memory.ttl，使用 tl:memoryOf + tl:outcome + tl:lessons 谓词，SPARQL 可检索

# ─── H7: Report ───
class RoundSummary(BaseModel):
    """Monitor → Main/Human 的轮次报告"""
    schema_version: str = "1.0"
    producer_agent: str = "main"
    interaction_id: str
    round: int
    timestamp: datetime
    tasks_total: int
    tasks_passed: int
    tasks_failed: int
    critic_stats: dict    # {"reviewer_pass_rate": 0.8, "tester_pass_rate": 0.7, ...}
    tokens_consumed: int
    duration_sec: float
    alerts: list[str]
```

**Schema 版本兼容性契约：**

```
MAJOR.MINOR 版本规则：
  - MAJOR 变更：删除了必填字段、改变了字段语义 → consumer 拒绝解析，报错
  - MINOR 变更：新增了 optional 字段 → consumer 忽略未知字段，兼容解析

consumer 校验逻辑：
  if deliverable.schema_version.major != expected.major:
      raise IncompatibleSchemaError(...)
  # minor 不同 → 忽略新字段，提取已知字段
```

交接物流向详见 §3.8.4 主循环流程定义中的 flows（f1-f12）。7 种交接物 Schema 定义如下：

**交接物验证规则（consumer 端必检）：**

| 检查项 | 失败处理 |
|--------|---------|
| `schema_version` major 兼容 | 拒绝消费，返回 IncompatibleSchemaError |
| `interaction_id` 匹配当前 loop | 拒绝消费，记录 WARN |
| 必填字段非空 | 拒绝消费，返回 ValidationError |
| `timestamp` 不早于 loop 启动时间 | 接受但记录 WARN（时钟偏差） |
| `producer_agent` 是合法的 agent ID | 拒绝消费，返回 UnknownAgentError |

---

#### 3.7.4.1 每个角色的模板规范

每个 agent 的输入输出均在 §3.1.2 契约表中定义。以下为 6 个角色的标准模板与产出文档。

##### coder — 执行模板

```
输入: TaskUnit {
  task: { task_id, description, verification },
  context: { goal, knowledge, past_failures }
}

产出文件: tests/test_*.py 或 src/**/*.py（按 KR 指定）
产出文档: TaskResult { output, status, reasoning, files_changed, metadata }

reasoning 模板:
  "为 ${module} 的 ${function} 编写了 ${test_count} 个测试。
   测试方法: ${HTTP请求/直接调用/...}。
   覆盖分支: ${covered_branches}。
   遇到问题: ${issues|无}。"
```

##### reviewer — 审查模板

```
输入: TaskResult
不可用工具: exec, write

审查产出: Verdict { lens: "correctness", passed, reasoning, issues, improvements }

审查清单（§3.6.6 完整定义）:
  ☐ 代码与 task spec 一致性
  ☐ 逻辑错误
  ☐ API 误用
  ☐ import 正确性
  ☐ 与已有代码兼容性
```

##### tester — 测试模板

```
输入: TaskResult
可用工具: read, exec
不可用工具: write (源文件)

测试产出: Verdict { lens: "completeness", passed, reasoning, issues, improvements }
         + CoverageReport { file, baseline_coverage, new_coverage, delta }

测试步骤:
  1. 环境隔离: 干净环境运行（不依赖其他测试副作用）
  2. pytest 收集+执行: 验证测试可运行
  3. coverage 测量: 对比基线，计算增量
  4. 回归检查: 全量测试套件通过
  5. flaky 检测: 同一测试连续 3 次运行结果一致
```

##### auditor — 审计模板

```
输入: TaskResult + reviewer.Verdict + tester.Verdict + tester.CoverageReport
不可用工具: exec, write

审计产出: Verdict { lens: "robustness", passed, reasoning, issues, improvements }

审计清单（§3.6.6 完整定义，12 项）:
  ☐ 错误处理完整性      ☐ reviewer 审查完整性
  ☐ 资源管理            ☐ tester 隔离完整性
  ☐ 注入安全            ☐ 交接物 Schema 合规
  ☐ CR 记录完整性       ☐ 日志可追溯性
  ☐ 流程合规            ☐ 模型使用合规
  ☐ 不执行代码(自检)    ☐ 不修改文件(自检)
```

##### publicist — 记忆模板

```
输入: TaskResult + Decision
可用工具: write
不可用工具: exec

记忆产出: MemoryEntry {
  task_id, task_summary, outcome, lessons, keywords
}
产出: rdf/ast/memory.ttl（Turtle 三元组，SPARQL 可检索）

outcome 模板:
  通过: "KR ${task_id} 通过 — ${简要描述}"
  未通过: "KR ${task_id} 未通过 — ${否决原因} — 建议: ${改进方向}"
```

##### planner — 规划模板

```
输入: StandardGoal + memory + KS查询结果
可用工具: LLM, read
不可用工具: exec, write

规划产出: TaskDAG { tasks: [{ task_id, description, priority, dependencies, verification }] }

KR 描述模板:
  "为 ${module} 的 ${function} ${正常路径/异常路径/边界条件} 编写 ${HTTP测试/单元测试}"
  
verification 模板:
  { method: "pytest", criteria: ["测试可运行", "覆盖率增量 ≥ +1%"], evidence: "coverage.json" }
```

---

#### 3.7.5 版本化变更管理：从文档到交付物的完整追溯链

**关注点分离：** 变更记录（CR）与变更执行分离。CR 记录"改了什么、为什么改、谁审查的"——这是记录层的职责。代码 diff、文件修改——这是执行层的职责。记录层不修改代码，执行层不篡改记录。

**CR 标准模板（每条变更的不可变记录）：**

```json
{
  "cr_id": "CR-20260628-001",
  "interaction_id": "manual-20260628T110000-a3f2",
  "status": "closed",
  "trigger": {
    "type": "critic_veto",
    "agent": "auditor",
    "task_id": "manual-20260628T110000-a3f2-L-R3-t5",
    "reason": "错误处理路径未覆盖异常分支"
  },
  "change": {
    "type": "test_addition",
    "files": ["tests/test_handlers.py"],
    "description": "新增 test_handle_tree_missing_ctx_key",
    "diff_lines": "+15"
  },
  "executor": {
    "agent": "coder",
    "timestamp": "2026-06-28T11:05:00Z"
  },
  "verification": {
    "reviewer": {"passed": true, "issues": []},
    "tester": {"passed": true, "coverage_delta": "+0.8%"},
    "auditor": {"passed": true, "checklist_items": 12}
  },
  "version": {
    "before": "v1.2.3",
    "after": "v1.2.4",
    "segment": "PATCH"
  },
  "closed_at": "2026-06-28T11:08:00Z"
}
```

**变更冲突解决规则：**

当同一文件的多个 CR 冲突时（coder 独立执行多个 KR 可能修改同一文件）：

```
冲突检测：
  CR-A 修改 tests/test_handlers.py:150-165
  CR-B 修改 tests/test_handlers.py:158-170
  → 重叠区域: 158-165 → 冲突

解决规则（按优先级）：
  1. 不同函数 → 自动合并（追加到文件末尾，不覆盖）
  2. 同一函数不同位置 → 自动合并（插入不重叠的行）
  3. 同一函数同一位置 → 人工介入（loop 暂停，通知人类）
  4. CR 之间有依赖关系 → 按 DAG 顺序执行，后执行的 CR 为最终版本
```

**回滚契约：**

```
CR 回滚不删除原 CR，而是创建逆向 CR：

CR-20260628-001: 新增 test_handle_tree_missing_ctx_key (PATCH v1.2.3→v1.2.4)
CR-20260628-005: 回滚 CR-001 (PATCH v1.2.4→v1.2.5)
  原因: test_handle_tree_missing_ctx_key 连续 3 次 flaky
  关联: reverts CR-20260628-001

回滚规则：
  - 原 CR 状态改为 "reverted"，不可再修改
  - 回滚 CR 引用原 CR，形成完整变更链
  - 版本号正常推进（回滚也是变更）
```

**版本号规则（与交接物联动）：**

| 版本段 | 触发条件 | 涉及交接物 | 示例 |
|--------|---------|-----------|------|
| **MAJOR** | H1(Goal) 变更 — O 级改动 | H1, H2, H3 | "覆盖率50%"→"覆盖率80%" |
| **MINOR** | H2(Task[]) 变更 — KR 新增/删除 | H2, H3, H7 | 新增 K2.4 |
| **PATCH** | H4(Delivery) 变更 — 文件修改 | H4 | 测试文件增删改 |

**CR 自动触发规则：**

CR 不需要人类手动创建。以下事件自动生成 CR：

```json
{
  "cr_auto_triggers": {
    "description": "当以下事件发生时，系统自动创建CR记录 [无需人工干预]",
    "rules": [
      {
        "event": "coder.write_file",
        "condition": "file_path matches 'tests/**' or 'src/**'",
        "action": "create_cr",
        "cr_type": "code_change",
        "description": "coder修改了项目文件时自动创建CR"
      },
      {
        "event": "critic.verdict_failed",
        "condition": "verdict.passed == false",
        "action": "create_cr",
        "cr_type": "fix_request",
        "description": "任意critic否决时创建修复CR，关联veto原因"
      },
      {
        "event": "planner.replan",
        "condition": "task_list_changed == true",
        "action": "create_cr",
        "cr_type": "plan_change",
        "description": "PlannerAgent重规划导致KR变更时创建CR"
      },
      {
        "event": "loop.terminated",
        "condition": "result in ['FATAL', 'MAX_ROUNDS', 'BUDGET', 'STAGNATION']",
        "action": "create_cr",
        "cr_type": "termination_record",
        "description": "非正常终止时创建终止记录CR，含diagnostic信息"
      }
    ]
  }
}
```

**CR 生命周期状态机：**

```
open → in_progress → under_review → closed
  │                     │
  └─────────────────────┘ (veto: 退回重做)
  │
  └── reverted (closed后创建逆向CR回滚)
```

**CR 编号规则：** `CR-{interaction_id_short}-{seq}`，如 `CR-a3f2-001`。

---

#### 3.7.6 五项基础设施的依赖关系与接口优先构建顺序

**关注点分离总结：** 五项基础设施共享同一条设计主线——采集与加工分离、生产与消费分离、机制与策略分离。这条主线确保了系统在复杂度增长时不会失控：每个模块只关心自己的一层，跨层交互通过明确的接口契约。

**构建顺序（接口优先，实现后置）：**

```
Step 1: 定义全部接口契约（1 天，不写实现）
  ├── core/interfaces.py     ← 6 个核心接口（纯 ABC，无实现）
  ├── core/types.py          ← 7 个交接物 Schema（纯 Pydantic）
  ├── core/events.py         ← 事件类型枚举
  └── [测试] 接口一致性检查：所有接口方法签名可 import 且无循环依赖

Step 2: 实现日志框架 + DI 容器（地基，0.5 天）
  ├── 3.7.1 日志框架 → 三个适配器（JsonLine/人类/指标）
  ├── 3.7.2 DI 容器 → 配置文件解析 + 接口注册
  └── [测试] 日志五级输出 + DI 容器装配 6 个 mock 接口

Step 3: 实现 Mock 全套（测试先行，0.5 天）
  ├── MockLLMProvider, MockKnowledgeProvider, MockAgentAdapter
  ├── MemoryStore, MemoryLogger, MemoryEventBus
  └── [测试] 用 mock 全套运行一个虚拟 loop（无真实 LLM 调用）

Step 4: 实现核心逻辑（积木搭建，0.5 天）
  ├── GoalAgent, PlannerAgent, LoopController
  ├── CriticAgent (3 视角), MemoryAgent, MonitorAgent
  ├── 3.7.3 架构隔离 → 核心逻辑层只依赖 interfaces.py
  └── [测试] 全 mock 环境下，1 轮完整 loop 运行通过

Step 5: 实现适配器（对接真实环境，0.5 天）
  ├── OpenClawAdapter, DeepSeekProvider, RDFServerAdapter
  ├── JsonFileStore, JsonLineLogger, HumanReadableLogger
  └── [测试] 切换配置为生产模式，真实环境 1 轮 loop

Step 6: 变更管理 + 标准交接物集成（0.5 天）
  ├── 3.7.5 CR 记录 → 每次文件变更自动生成 CR
  ├── 3.7.4 交接物校验 → consumer 端 Schema 校验
  └── [测试] 端到端：/goal → 多轮执行 → CR 链完整可追溯
```

组件间依赖关系详见 §3.8.4 主循环流程定义中的 roles 定义和 flows 信息流。构建顺序与验收标准如下：

**接口优先的测试策略：**

接口定义完成 → 立即写测试。实现还没写之前，测试就已经验证了接口的可用性：

```
Step 1 完成 → 测试：所有接口可 import、无循环依赖、方法签名合理
Step 2 完成 → 测试：日志五级输出、DI 容器可装配 6 个 mock
Step 3 完成 → 测试：虚拟 loop（全 mock）1 轮完成
Step 4 完成 → 测试：核心逻辑正确性（Mock 环境）
Step 5 完成 → 测试：适配器对接真实环境
Step 6 完成 → 端到端：交互编号贯穿、CR 链完整、交接物 Schema 校验
```

**"积木"验收标准：**

每个 Step 完成后，对应模块应满足：
1. **接口符合契约** — 实现的方法签名与接口定义完全一致
2. **单元测试独立通过** — 不依赖其他 Step 的产物
3. **替换实现不影响上游** — 切换 mock↔生产，上游代码不变
4. **日志完整可追溯** — 按 interaction_id 筛选可看到该 Step 的全部事件

### 3.8 角色×步骤矩阵：roles-steps-flows 流程描述规约

team-loop 所有 agent 协作流程使用统一的 roles-steps-flows 格式描述。该格式以二维表（列=角色、行=步骤序号）为主体，以有向边描述信息流，替代难以维护的 ASCII 框线图。

#### 3.8.1 格式定义

```json
{
  "format": "roles-steps-flows",
  "version": "1.0",
  "description": "用二维表(角色×步骤)+有向边描述agent协作流程，列=角色，行=步骤序号，单元格=任务，单元格间的有向边=信息流",
  "roles": {
    "description": "参与本次协作的角色列表，每个角色定义其id和职责描述",
    "items": [
      {
        "id": "main",
        "description": "主控agent，负责目标解析、任务规划、调度决策、终止判断"
      }
    ]
  },
  "steps": {
    "description": "按seq序号排列的步骤。同一seq下可有多个角色并行执行（cells中多个key），不同seq间默认串行，flows中可定义跨seq依赖",
    "items": [
      {
        "seq": 1,
        "description": "本步骤的整体描述，说明此步骤的目标",
        "cells": {
          "main": {
            "task": "任务简述",
            "description": "任务的详细描述，包括做什么、为什么做",
            "in": [],
            "out": ["StandardGoal"]
          }
        }
      }
    ]
  },
  "flows": {
    "description": "步骤间和角色间的数据流向。每条flow用from/to坐标指源和目标，what指明传递的数据类型",
    "items": [
      {
        "id": "f1",
        "from": {"role": "main", "seq": 1},
        "to": {"role": "main", "seq": 2},
        "what": "StandardGoal",
        "description": "GoalAgent产出的结构化目标传递给PlannerAgent做任务拆解"
      }
    ]
  }
}
```

#### 3.8.2 字段约束

本格式与 ZenUML 时序图 DSL 一一对应。可选补充说明（超时、条件、子流程引用等）不设独立字段，直接拼接到 `description` 文本中，用 `[...]` 标识为可选部分。

##### 根级字段

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `format` | ✅ | `"roles-steps-flows"` | 固定值 |
| `version` | ✅ | `"MAJOR.MINOR"` | 格式版本号 |
| `description` | ✅ | string | 流程整体说明 |

##### roles（参与者）

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `roles.description` | ✅ | string | 角色列表说明 |
| `roles.items[].id` | ✅ | string | 角色标识 |
| `roles.items[].description` | ✅ | string | 角色职责描述 |

##### steps（步骤矩阵 → ZenUML 时间轴）

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `steps.description` | ✅ | string | 步骤序列说明 |
| `steps.items[].seq` | ✅ | int | 步骤序号，从 1 递增 |
| `steps.items[].description` | ✅ | string | 步骤整体描述。可选补充用 `[...]` 标注 |
| `steps.items[].cells.{role_id}` | ✅ | object | 角色在本步骤的任务 |
| `cells.{role}.task` | ✅ | string | 任务简述 |
| `cells.{role}.description` | ✅ | string | 任务详细描述。可选补充用 `[...]` 标注 |
| `cells.{role}.in` | ✅ | string[] | 输入数据类型 |
| `cells.{role}.out` | ✅ | string[] | 输出数据类型 |

##### flows（消息 → ZenUML `A->B: what` / `A->>B: what`）

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `flows.description` | ✅ | string | 数据流说明 |
| `flows.items[].id` | ✅ | string | 流唯一标识 |
| `flows.items[].type` | ✅ | `"sync"` \| `"async"` | sync=`->`（同角色内），async=`->>`（跨角色） |
| `flows.items[].from` | ✅ | {role, seq} | 源坐标 |
| `flows.items[].to` | ✅ | {role, seq} | 目标坐标 |
| `flows.items[].what` | ✅ | string | 传递的数据类型名 |
| `flows.items[].description` | ✅ | string | 文字说明。可选补充用 `[...]` 标注 |



#### 3.8.3 语义规则

**并行规则：** 同一 `seq` 下有多个 cell → 并行执行。对应 ZenUML `par { ... }`。

**串行规则：** 不同 `seq` 默认串行。`flows` 中 `from.seq > to.seq` 为反馈流（如 critic 反馈回 planner），对应 ZenUML 中消息箭头指向上方。

**消息类型（sync/async）：**

| 场景 | type | ZenUML | 原因 |
|------|------|--------|------|
| 同一角色内 seq 间传递 | `sync` | `A->A:` | 同一 agent 进程内函数调用 |
| 跨角色派发任务 | `async` | `A->>B:` | sessions_spawn，fire and forget |
| 跨角色返回结果 | `async` | `A->>B:` | push event 返回 |

**输入输出校验：** 每条 flow `(A→B, what=X)` 必须满足 `X ∈ A.cell.out` 且 `X ∈ B.cell.in`。


**`[...]` 中括号标注规范：**

`[...]` 是 `description` 文本的一部分，标识括号内的内容为可选补充说明。括号内是自然语言，不是 key:value 格式。正文在前，可选补充在后，空格分隔。

示例：
```
"下发TaskUnit并等待执行完成 [超时300秒] [子流程:krexec-loop]"
"裁决结果输入终止判断 [仅当通过票数≥2]"
"主循环反复执行，每轮重新规划 [直到终止条件触发或轮次耗尽]"
"三视角并行验证 [超时120秒]"
"基础层连续失败，跳出主循环 [预算耗尽或KS不可达]"
```

**语义约定（括号内容为描述，无固定 key）：**

| 常见内容 | 放置位置 | UML 依据 | 示例 |
|---------|---------|---------|------|
| 超时 | flows.description | UML DurationConstraint 属于消息 | `[超时300秒]` |
| 子流程引用 | steps.description | UML InteractionUse(ref) | `[子流程:krexec-loop]` |

#### 3.8.3.1 通用流契约（所有 flow 的基类约束）

每条 flow 无论其具体内容，必须遵守以下 IPO 契约。类比面向对象：这是所有 flow 的抽象父类，§3.8.4 中的 f1-f12 是具体子类实例。

##### I — 输入约束（flow 发送前校验）

| # | 约束 | 校验方式 | 违反处理 |
|---|------|---------|---------|
| I1 | `from.role` 存在于 `roles.items` 中 | 查表 | 拒绝发送，FATAL |
| I2 | `to.role` 存在于 `roles.items` 中 | 查表 | 拒绝发送，FATAL |
| I3 | `from.seq` 对应 cell 的 `out` 数组包含 `what` | 查 steps[from.seq].cells[from.role].out | 拒绝发送，WARN |
| I4 | `to.seq` 对应 cell 的 `in` 数组包含 `what` | 查 steps[to.seq].cells[to.role].in | 拒绝发送，WARN |
| I5 | `from.seq` < `to.seq` 或显式标注为反馈流 | 比较序号 | 允许但 WARN（反馈流） |
| I6 | 发送方 agent 已就绪（非 FATAL 状态） | 查 agent 状态 | 等待或跳过 |

##### P — 处理约束（flow 传输中行为）

| # | 约束 | 说明 |
|---|------|------|
| P1 | `type: sync` → 发送方阻塞，直到接收方处理完毕并返回 | 同角色内函数调用 |
| P2 | `type: async` → 发送方立即继续，接收方异步处理，完成后通过另一条 flow 回复 | 跨角色 agent 通信 |
| P3 | 跨角色 flow 必须为 async（agent 间无同步调用） | 架构约束 |
| P4 | 每条 flow 携带隐式 `interaction_id`、`loop_id`、`round`、`timestamp` | 日志追溯 |
| P5 | 携带的数据必须符合 §3.7.4 定义的 Schema | Schema 校验 |

##### O — 输出约束（flow 接收后行为）

| # | 约束 | 说明 |
|---|------|------|
| O1 | 接收方必须校验数据 Schema | consumer 端 §3.7.4 校验 |
| O2 | 接收方处理完成后，若有回复，必须通过另一条 flow 返回 | 形成请求-响应对 |
| O3 | 接收方处理超时 → 触发 `[超时]` 标注的秒数后 WARN | 防止 agent 卡死 |
| O4 | 接收方处理异常 → ERROR 日志，不静默 | 可追溯 |

##### 流校验时序

```
发送前                   传输中                  接收后
───────                 ───────                 ──────
I1-I6 校验              P1-P5 行为              O1-O4 行为
  │                       │                       │
  ├─ 角色存在？            ├─ sync阻塞/async继续    ├─ Schema校验
  ├─ in/out匹配？          ├─ 跨角色强转async       ├─ 超时检测
  ├─ 序号合法？            ├─ 隐式字段注入           ├─ 异常不静默
  └─ 发送方可用？          └─ Schema携带            └─ 回复流关联
```

##### 通用约束速查

所有 flow 实例（f1-f12）自动继承以上约束，无需逐条声明。§3.8.4 实例中各 flow 的 `type`、`from`、`to`、`what`、`description` 是子类特化字段，其余均来自本基类契约。

#### 3.8.4 实例：team-loop 主循环流程

```json
{
  "format": "roles-steps-flows",
  "version": "1.0",
  "description": "team-loop 一次KR执行的标准流程。每轮执行1个KR，可跨轮逐步推进。auditor在reviewer和tester完成后独立审计，非并行。",
  "roles": {
    "description": "参与主循环的6个agent角色。main是中枢，承担GoalAgent/PlannerAgent/LoopController/Monitor四重角色。",
    "items": [
      {"id": "main",     "description": "主控agent。作为GoalAgent调用LLM解析目标。作为PlannerAgent调用LLM拆解KR。作为LoopController纯程序化调度判决。作为Monitor纯程序化聚合指标。"},
      {"id": "coder",    "description": "执行agent。按TaskUnit编写代码或生成测试。完成后产出TaskResult含reasoning自述。"},
      {"id": "reviewer", "description": "correctness critic。静态审查代码正确性，不运行代码。产出Verdict。"},
      {"id": "tester",   "description": "completeness critic。动态执行测试，测量覆盖率，负责环境隔离。产出Verdict+CoverageReport。"},
      {"id": "auditor",  "description": "robustness critic。接收reviewer和tester的Verdict，审计审查者有没有审查到位。永不执行代码。"},
      {"id": "publicist","description": "memory agent。记录KR执行结果到rdf/ast/memory.ttl，供下轮PlannerAgent查询。"}
    ]
  },
  "steps": {
    "description": "一次KR执行的10个步骤。每轮执行1个KR，按priority排序。与§2.5契约伪代码一致。",
    "items": [
      {
        "seq": 1,
        "description": "解析人类目标为结构化StandardGoal",
        "cells": {
          "main": {
            "task": "解析目标",
            "description": "将人类自然语言目标转化为结构化StandardGoal，提取约束、成功标准、优先级。调用LLM。",
            "in": [],
            "out": ["StandardGoal"]
          }
        }
      },
      {
        "seq": 2,
        "description": "查询KS获取项目上下文，结合Memory拆解为KR任务DAG。调用LLM。",
        "cells": {
          "main": {
            "task": "查询KS + 规划任务",
            "description": "向KS查询项目运行时上下文和测试范例；查询Memory获取历史记忆和失败记录。将Objective拆解为三层KR（基础层→覆盖层→质量层），生成TaskDAG。每个KR含verification字段和max_retries字段(默认2)。",
            "in": ["StandardGoal"],
            "out": ["TaskDAG"]
          }
        }
      },
      {
        "seq": 3,
        "description": "主控取最高priority的就绪KR，下发TaskUnit给coder。纯程序化，不调用LLM。",
        "cells": {
          "main": {
            "task": "下发TaskUnit",
            "description": "从TaskDAG中取最高priority且依赖已满足的就绪KR。每轮1个KR。装配TaskUnit（含上下文refs），通过sessions_spawn下发给coder。",
            "in": ["TaskDAG"],
            "out": ["TaskUnit"]
          }
        }
      },
      {
        "seq": 4,
        "description": "coder执行任务",
        "cells": {
          "coder": {
            "task": "执行任务",
            "description": "解析TaskUnit，按需查询KS获取上下文，编写代码或生成测试。完成后产出TaskResult含reasoning自述。",
            "in": ["TaskUnit"],
            "out": ["TaskResult"]
          }
        }
      },
      {
        "seq": 5,
        "description": "main收集TaskResult并分发给reviewer和tester。纯程序化。",
        "cells": {
          "main": {
            "task": "收集结果并分发",
            "description": "等待coder完成，收集TaskResult。分发给reviewer和tester进行并行验证。auditor暂不分发——需等reviewer和tester产出后审计。",
            "in": ["TaskResult"],
            "out": ["TaskResult", "TaskResult"]
          }
        }
      },
      {
        "seq": 6,
        "description": "reviewer和tester并行验证。auditor在此seq等待两者的产出。",
        "cells": {
          "reviewer": {
            "task": "正确性静态审查",
            "description": "阅读代码diff，对比task spec，查找逻辑错误、API误用、设计缺陷。不运行代码。按§3.6.6审查清单逐项检查。",
            "in": ["TaskResult"],
            "out": ["Verdict"]
          },
          "tester": {
            "task": "完整性动态执行",
            "description": "在隔离环境中运行测试，测量覆盖率，验证所有task需求是否有测试覆盖。产出CoverageReport。按§3.6.6审查清单逐项检查。",
            "in": ["TaskResult"],
            "out": ["Verdict", "CoverageReport"]
          }
        }
      },
      {
        "seq": 7,
        "description": "auditor接收reviewer和tester的Verdict+报告，独立审计两者有没有审查到位",
        "cells": {
          "auditor": {
            "task": "健壮性审计",
            "description": "审阅reviewer的审查报告和tester的测试报告+CoverageReport。检查reviewer是否遗漏关键逻辑路径，tester是否在真实隔离环境运行。按§3.6.6审计清单12项逐项检查。永不执行代码。",
            "in": ["Verdict", "Verdict", "CoverageReport"],
            "out": ["Verdict"]
          }
        }
      },
      {
        "seq": 8,
        "description": "main收集三个verdict，做2/3裁决。纯程序化。",
        "cells": {
          "main": {
            "task": "2/3裁决",
            "description": "收集reviewer+tester+auditor三个verdict。统计passed≥2则KR通过；passed≤1且retry_count<task.max_retries则重试(回seq2)；passed≤1且retry_count>=task.max_retries则跳过。跳过时标记此KR为SKIPPED，DAG拓扑中视为已满足以允许下游继续。",
            "in": ["Verdict", "Verdict", "Verdict"],
            "out": ["Decision"]
          }
        }
      },
      {
        "seq": 9,
        "description": "终止条件判断。纯程序化。",
        "cells": {
          "main": {
            "task": "终止判断",
            "description": "按§3.4 check_order逐条检查终止条件。未终止时通过f13反馈回seq2继续下一轮（或同一轮中下一个KR）。终止时通过f14通知publicist产出最终报告。继续时组装LoopContext（含Decision+MemoryEntry+CoverageReport）传给下轮PlannerAgent。",
            "in": ["Decision"],
            "out": ["SUCCESS", "RETRY", "NEXT_KR", "FATAL", "MAX_ROUNDS", "BUDGET", "STAGNATION"]
          }
        }
      },
      {
        "seq": 10,
        "description": "publicist沉淀记忆",
        "cells": {
          "publicist": {
            "task": "记录Memory",
            "description": "将本轮KR执行结果（成功/失败、原因、教训）结构化记录到rdf/ast/memory.ttl。使用tl:memoryOf/tl:outcome/tl:lessons谓词。下轮PlannerAgent通过KS查询Memory。",
            "in": ["TaskResult", "Decision"],
            "out": ["MemoryEntry"]
          }
        }
      }
    ]
  },
  "flows": {
    "description": "主循环中16条数据流，含1条反馈环(f13)和1条记忆回流(f15)用于下一轮规划",
    "items": [
      {"id":"f1", "type":"sync", "from":{"role":"main","seq":1},"to":{"role":"main","seq":2},"what":"StandardGoal","description":"结构化目标传递给PlannerAgent"},
      {"id":"f2", "type":"sync", "from":{"role":"main","seq":2},"to":{"role":"main","seq":3},"what":"TaskDAG","description":"任务DAG传递给LoopController调度"},
      {"id":"f3", "type":"async", "from":{"role":"main","seq":3},"to":{"role":"coder","seq":4},"what":"TaskUnit","description":"主控下发任务单元给执行器 [超时300秒]"},
      {"id":"f4", "type":"async", "from":{"role":"coder","seq":4},"to":{"role":"main","seq":5},"what":"TaskResult","description":"coder返回交付物给主控"},
      {"id":"f5", "type":"async", "from":{"role":"main","seq":5},"to":{"role":"reviewer","seq":6},"what":"TaskResult","description":"主控分发交付物给正确性审查 [超时120秒]"},
      {"id":"f6", "type":"async", "from":{"role":"main","seq":5},"to":{"role":"tester","seq":6},"what":"TaskResult","description":"主控分发交付物给完整性验证 [超时120秒]"},
      {"id":"f7", "type":"async", "from":{"role":"reviewer","seq":6},"to":{"role":"auditor","seq":7},"what":"Verdict","description":"reviewer的正确性投票传给auditor供审计"},
      {"id":"f8", "type":"async", "from":{"role":"tester","seq":6},"to":{"role":"auditor","seq":7},"what":"Verdict","description":"tester的完整性投票传给auditor供审计"},
      {"id":"f9", "type":"async", "from":{"role":"tester","seq":6},"to":{"role":"auditor","seq":7},"what":"CoverageReport","description":"tester的覆盖率报告传给auditor供审计"},
      {"id":"f10","type":"async", "from":{"role":"auditor","seq":7},"to":{"role":"main","seq":8},"what":"Verdict","description":"auditor的robustness投票"},
      {"id":"f11","type":"async", "from":{"role":"reviewer","seq":6},"to":{"role":"main","seq":8},"what":"Verdict","description":"reviewer的correctness投票（用于2/3计数）"},
      {"id":"f12","type":"async", "from":{"role":"tester","seq":6},"to":{"role":"main","seq":8},"what":"Verdict","description":"tester的completeness投票（用于2/3计数）"},
      {"id":"f13","type":"sync", "from":{"role":"main","seq":8},"to":{"role":"main","seq":9},"what":"Decision","description":"2/3裁决结果输入终止判断"},
      {"id":"f14","type":"sync", "from":{"role":"main","seq":9},"to":{"role":"main","seq":2},"what":"LoopContext","description":"[反馈环] 未终止时触发下一轮。LoopContext含Decision+MemoryEntry+CoverageReport，PlannerAgent结合Memory和本轮决策调整规划"},
      {"id":"f15","type":"sync", "from":{"role":"main","seq":9},"to":{"role":"publicist","seq":10},"what":"Decision","description":"裁决结果供publicist记录"},
      {"id":"f16","type":"async", "from":{"role":"publicist","seq":10},"to":{"role":"main","seq":2},"what":"MemoryEntry","description":"publicist产出的记忆供下轮PlannerAgent查询。PlannerAgent在seq2通过KS查询rdf/ast/memory.ttl获取"}
    ]
  }
```

#### 3.8.5 使用场景

| 场景 | 用途 | 消费者 |
|------|------|--------|
| **文档** | 描述各模块的 agent 协作流程，替代 ASCII 框线图 | 人（开发者） |
| **流程校验** | 检查 flows 的 what 是否匹配上游 out 和下游 in | CI 脚本 |
| **调度配置** | LoopController 根据 steps 和 flows 生成执行计划 | LoopController |
| **交接物生成** | 从流程定义自动生成 TaskUnit 模板 | PlannerAgent |

## 四、可行性分析

### 4.1 技术可行性

| 组件 | 可行性 | 依据 |
|------|--------|------|
| Knowledge Server | ✅ 高 | rdflib + SPARQL，详见 projects/pyrdf-knowledge |
| Agent 角色复用 | ✅ 高 | MA 6 agent 直接映射 §3.1.2 契约，零配置新增。每 agent 的 in/out/tools 约束已定义 |
| 循环调度模式 | ✅ 高 | main /goal 新增触发路径 + §3.8.4 主循环流程已定义。不影响 团队研发 12 阶段瀑布 |
| 3 视角对抗验证 | ✅ 高 | reviewer(静态读)+tester(动态执行+隔离)+auditor(审计审查) 三手段不同，§3.6.6 审查清单已定义 |
| 记忆闭环 | ✅ 中 | publicist 结构化 MemoryEntry + Knowledge Server 索引 + PlannerAgent 查询。跨项目经验提炼需积累后增强 |
| 日志框架 | ✅ 高 | §3.7.1 三重输出设计，纯 Python 标准库，不引入新依赖 |
| 变更管理 | ✅ 高 | §3.7.5 CR 模板 + 自动触发规则 + 版本号规则 |
| 标准交接物 | ✅ 高 | §3.7.4 7 种 Pydantic Schema + consumer 端校验 |

### 4.2 资源评估

| 项目 | 数值 |
|------|------|
| **新增代码量** | ~1200 行（KS 400行 + 循环调度 300行 + 日志框架 200行 + DI 容器 150行 + CR/交接物 150行） |
| **新增 agent** | 0（全部复用 MA 现有配置） |
| **新增依赖** | rdflib（pyOV 已有）；pydantic（Schema 校验）；其他纯 stdlib |
| **开发周期** | 4-5 天（Step1-2 接口+日志+DI 1.5天 + Step3 Mock 0.5天 + Step4 核心逻辑 1天 + Step5-6 适配器+集成 1天 + 联调 1天） |
| **破坏现有功能** | 无（/goal 是新增触发路径，不影响 团队研发 模式） |

### 4.3 风险与缓解

| 风险 | 级别 | 缓解 |
|------|------|------|
| 无人值守 token 消耗失控 | 🟡 中 | §3.4 预算熔断 + 轮次上限；monitor 实时追踪 token 消耗 |
| CriticAgent 误判（2/3 机制失效） | 🟡 中 | coder≠reviewer≠tester 使用不同模型；auditor 审查 critic 们的审查质量 |
| RDF 查询性能 | 🟢 低 | 项目知识规模小（<1000 三元组） |
| 记忆膨胀 | 🟢 低 | 最近 N 条限制 + 去重 + 每轮只保留 MemoryEntry |
| agent 间通信超时 | 🟡 中 | flows 上标注超时；§3.4 agent_timeout 终止规则；看门狗进程 |

### 4.4 核心优势（相比独立构建）

| 维度 | 独立 team-loop | 基于 MA |
|------|---------------|---------|
| agent 配置 | 需新建 7 个 agent | 0 个，复用现有 MA |
| agent 契约 | 需定义 | 已有 §3.1.2 契约表 |
| 通信矩阵 | 需重新设计 | MA 已有 main↔all hub 模式 |
| 质量流程 | 需从零设计 | 已有 reviewer+tester+auditor 三视角 + §3.6.6 审查清单 |
| 流程追踪 | 需重新设计 | 已有 todo.md / journey.md |
| 双模型互补 | 需重新配置 | 已配置（coder≠reviewer≠tester） |
| 维护成本 | 维护两套 | 维护一套 |

## 五、实施计划

按 §3.7.6 接口优先构建顺序执行。

### Step 1：定义全部接口契约（1 天）

```
- core/interfaces.py ← 6 个核心接口（LLMProvider, KnowledgeProvider, AgentAdapter, StoreProvider, LoggerProvider, EventBus）
- core/types.py ← 7 个交接物 Pydantic Schema（§3.7.4）
- core/events.py ← 事件类型枚举
- 测试：接口可 import、无循环依赖、方法签名合理
```

### Step 2：日志框架 + DI 容器（0.5 天）

```
- 3.7.1 日志框架实现 → JsonLineLogger + HumanReadableLogger + MetricsLogger
- 3.7.2 DI 容器 → JSON/YAML 配置解析 + 接口注册 + 装配
- 测试：五级日志输出 + 容器装配 6 mock 接口
```

### Step 3：Mock 全套实现（0.5 天）

```
- MockLLMProvider, MockKnowledgeProvider, MockAgentAdapter
- MemoryStore, MemoryLogger, MemoryEventBus
- 测试：全 mock 环境运行虚拟 loop（无真实 LLM）
```

### Step 4：核心逻辑实现（1 天）

```
- GoalAgent (§3.5.8 prompt) + PlannerAgent (§3.5.8 prompt) + LoopController
- CriticAgent 三视角（reviewer/tester/auditor，§3.6.6审查清单）
- MemoryAgent (publicist) + MonitorAgent
- 3.7.3 架构隔离 → 核心逻辑层只依赖 interfaces.py
- 测试：全 mock 环境 1 轮完整 loop
```

### Step 5：适配器实现（0.5 天）

```
- OpenClawAdapter (sessions_spawn) + DeepSeekProvider + GLMProvider
- RDFServerAdapter + JsonFileStore
- 测试：切换配置为生产模式，真实环境 1 轮 loop
```

### Step 6：集成联调 + 端到端验证（1 天）

```
- pyOV 项目端到端：/goal "提高handler覆盖率到50%" → 完整loop执行
- §3.7.5 CR 自动触发 + §3.7.4 交接物 Schema 校验
- §3.7.1 日志三重输出全量验证
- MA 团队研发模式回归（不受影响）
- 测试：端到端交互链按 interaction_id 完整可追溯
```

**总工期：4-5 天。**

## 六、预期收益

### 6.1 以 pyOV 为基线

| 阶段 | 关键能力 | handler 通过率 | handler 覆盖率 |
|------|---------|---------------|---------------|
| 当前（debugloop） | 裸 LLM 生成，无上下文 | 0/3 (0%) | ~0% |
| Step 1-3 完成 | 接口契约 + Mock 全环境 | — | — [仅框架，无真实执行] |
| Step 4 完成 | Knowledge Server + 循环调度 + 三视角 | ≥1/3 (33%) | 10-20% |
| Step 5-6 完成 | 真实 LLM 对接 + 记忆闭环 | ≥2/3 (67%) | 30-50% |
| 持续运行 | 3 视角对抗验证全开 + KR 自适应 | 3-5/轮 | 50-70% |

### 6.2 通用收益

- **零人工介入**的持续质量守护：/goal 设定目标后系统自主循环
- **知识常驻**：Knowledge Server 常驻伺服，不每次重扫项目
- **失败即学习**：Memory + PlannerAgent 反馈环，不重复犯错
- **agent 能力复用**：零新 agent，MA 现有 6 agent 直接服役
- **接口优先**：§3.7.6 接口契约先行，测试即文档，实现即堆积木
- **全程可追溯**：interaction_id 串联决策链 + CR 版本化变更管理
- **工程级基础设施**：分级日志 + DI + 架构隔离 + 标准交接物 + 变更管理

---

## 七、附录

### A. 与 debugloop-context-injection-analysis.md 的关系

本立项书是该分析文档的自然演进：

- 分析文档诊断了「信息不对称」问题
- 本方案将解决方案从「AST 提取 + prompt 注入」升级为「Knowledge Server + SPARQL 查询」
- 将范围从「补测试覆盖率」扩展为「通用自主任务执行」

### B. 参考材料

- `debugloop-context-injection-analysis.md` — 问题诊断与原始方案
- `reference/loopengineering/` — Loop Engineering 开源实现
- `reference/articles/loop-engineering-guide-lushbinary.md` — Loop Engineering 完整指南
- MA 架构：`projects/ma/multi-agent-design.md`
- MA 时序：`projects/ma/sequence-diagram.md`

### C. OpenClaw sessions_spawn 对接规范

team-loop 所有跨 agent 通信（main→coder、coder→main、main→critic、critic→main）均通过 OpenClaw `sessions_spawn` 实现。以下为与 OpenClaw runtime 的对接约束。

#### C.1 核心规则

| # | 规则 | OpenClaw 文档依据 | team-loop 对应 |
|---|------|-----------------|---------------|
| 1 | `sessions_spawn` 非阻塞，立即返回 `runId` + `childSessionKey` | docs.openclaw.ai | §3.8.4 所有跨角色 flow 的 type 为 async |
| 2 | 禁止 poll 循环等待，必须用 `sessions_yield` | docs.openclaw.ai | main 下发 TaskUnit 后 yield，收到 completion event 后继续 |
| 3 | context 默认 `isolated`，非 `fork` | docs.openclaw.ai | 与 Ralph 技法一致——每轮 fresh context |
| 4 | 子 session 工具受限，leaf agent 无 session 工具 | docs.openclaw.ai | coder/reviewer/tester/auditor 不能 spawn 孙 agent，符合 team-loop 深度=1 的设计 |
| 5 | 必须设置 `timeoutSeconds` | stanza.dev best practices | §3.8.4 f3 [超时300秒]、f5-f7 [超时120秒] |
| 6 | 并行子 agent 通过 `sessions_yield` 集中等待 | docs.openclaw.ai | §3.8.4 seq 6 三视角并行 → main 一次 yield 等 3 个 completion event |
| 7 | 完成后自动 announce 结果到 requester channel | docs.openclaw.ai | coder 产出 TaskResult → push event 回 main → main 收集后分发给 critic |

#### C.2 main→子 agent 的 spawn 调用

```json
// main 下发 TaskUnit 给 coder
{
  "tool": "sessions_spawn",
  "agentId": "coder",
  "task": "为 ontology_handler.py 的 handle_tree 正常路径编写 HTTP 测试",
  "taskName": "t5-handle-tree-test",
  "context": "isolated",
  "runtime": "subagent",
  "runTimeoutSeconds": 300
}
```

### D. 项目目录结构

team-loop 管理的项目标准目录布局。遵循最佳实践，新增 `reference/`（参考资料）和 `examples/`（示例代码/数据），按源码→测试→文档→参考→示例→脚本→前端→抽象接口→适配器→容器→配置→日志→产物→知识→CI/CD→构建工具→元数据→入口文件顺序排列。

```
project/                          ← 被 team-loop 管理的项目
├── src/                          ← 编译源码
├── tests/                        ← 测试案例
├── docs/                         ← 文档
├── reference/                    ← 参考资料（规格、协议、设计文档等）
├── examples/                     ← 示例代码/数据
├── scripts/                      ← sh/py/js 脚本
├── web/                          ← 前端代码 (js/rc/css)
├── core/                         ← 抽象接口（§3.7.2 的 6 个 ABC）
│   └── interfaces.py
├── adapters/                      ← 接口实现（对接具体框架）
│   ├── openclaw.py                ← AgentAdapter
│   ├── deepseek.py                ← LLMProvider
│   └── rdf.py                     ← KnowledgeProvider
├── container.py                   ← DI 容器（读 config 装配依赖）
├── config/                        ← 配置
│   └── bindings.json              ← 接口→实现的绑定（测试/生产切换）
├── logs/                         ← 日志落盘（§3.7.1 三重输出）
├── bin/                          ← 编译产物
├── rdf/                          ← 项目知识（详见 projects/pyrdf-knowledge）
├── .github/
│   └── workflows/
│       ├── test.yml               ← CI：跑测试
│       └── lint.yml               ← CI：代码检查
├── Makefile                       ← make test / make build
├── .team-loop/                   ← 过程文档（不提交 git）
│   ├── STATE.md                  ← 唯一的状态文档。内容：项目累计费用 + 当前loop费用 + 最近一次执行 + 状态
│   ├── config.json               ← 项目配置（预算、轮次、模型）
│   ├── cr/                       ← 变更记录（§3.7.5）
│   └── reports/                  ← 轮次报告（§3.7.4 H7）
├── README.md                     ← 项目说明
└── .gitignore                    ← 含 .team-loop/
```

**目录规则：**
- **排序逻辑：** 源码→测试→文档→参考→示例→脚本→前端→抽象接口→适配器→容器→配置→日志→产物→知识→CI/CD→构建工具→元数据→入口文件
- `reference/` — 参考资料，如项目相关的规格、协议、外部设计文档等
- `examples/` — 示例代码和数据，供理解项目用法
- `rdf/` 分两类：`business/`（人工/agent 沉淀的业务知识）和 `ast/`（KS 自动扫描源码提取）
- `.team-loop/STATE.md` — 唯一状态文档，while 循环 grep 此文件。内容：running / SUCCESS / FATAL / MAX_ROUNDS / BUDGET / STAGNATION
- `.team-loop/` — 不提交到项目 git repo（.gitignore）
- `rdf/` — 项目本地 RDF 知识文件，唯一键 = `项目名 + rdf文件名`。KS 服务按需增删改查
- `logs/` — §3.7.1 三重输出（JSON 行 + 人类可读 + 指标）均落盘于此

**全局知识库（跨项目通用，非项目目录内）：**
```
~/.openclaw/kb/team-loop/
├── patterns/       ← 通用模式（如 "FastAPI handler 测试走 HTTP 请求"）
├── llm-behaviors/  ← LLM 行为知识（如 "DeepSeek thinking=disabled 指令遵循下降"）
└── anti-patterns/  ← 失败模式库（如 "直接构造ctx导致setup崩溃"）
```

---

> 立项依据来自 2026-06-27 晚间团队的完整讨论。
> 核心方法论：不翻译，喂原材 | 知识伺服 | maker/checker 分离 | 对抗验证 | 记忆闭环 | Ralph 原子化
