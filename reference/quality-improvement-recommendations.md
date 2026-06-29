# debugloop 项目质量改进建议

> 综合 debugloop 上下文注入分析 + Loop Engineering 五构建块（+记忆）框架
>
> 编制日期：2026-06-27

---

## 概述

本建议基于两份材料：

- **材料A**：`debugloop-context-injection-analysis.md` — pyOV 项目实战分析。debugloop 基线覆盖率 33.9%，handler 层 942 个分支覆盖率 0%，3 个 LLM 生成测试全部在 setup 阶段崩溃。根因为"信息不对称"——LLM 缺少项目运行时上下文的"装配图"。
- **材料B**：Loop Engineering 框架 — Automations（心跳）、Worktrees（隔离）、Skills（项目知识）、Plugins（工具集成）、Sub-agents（maker/checker 分离）、Memory（外部记忆），以及对抗验证（3 视角 2/3 通过）和 Ralph 技法（fresh context + 单元工作 + commit + 退出）。

以下建议按"debugloop 具体问题 → Loop Engineering 对应原则 → 可执行步骤"的结构组织，每条标注优先级（P0/P1/P2）和预期收益。

---

## 建议 1：项目知识注入机制 — Skills 化

### 1.1 映射到 debugloop 具体问题

**核心问题（材料A §1.2）：** LLM 收到散落的零件（源码/签名/导入路径），但缺"装配图"——不知道 `ctx` 里该放什么、不知道项目的运行时初始化模式、不知道已有测试的正确写法是什么。

**具体表现：**
- pyOV handler 函数签名 `handle_tree(query, body, ctx)`，LLM 知道签名但不知道 `ctx` 的真实结构（`{"store": TripleStore, "tbox": TBoxEngine, ...}`）
- 已有测试文件 `test_handlers.py`（800+ 行）未被作为范例注入
- `conftest.py` 只做了路径设置，未提供 fixture，但 `run_server.py` 包含完整的初始化逻辑——LLM 看不到
- 3 个生成的测试全部在 setup 阶段崩溃，非断言失败

**debugloop 的现有能力缺口（材料A §2.3）：**

| 现有能力 | 缺口 |
|----------|------|
| API 签名清单 | 不包含模块间依赖关系 |
| 项目导入路径 | 不包含运行时构造方式 |
| 设计文档读取 | 简单截断，不提取关键信息 |
| 函数调用示例 | 不包含测试文件中的用法 |
| 函数源码注入 | 不含外部依赖上下文 |

### 1.2 引用 Loop Engineering 原则

**Skills（项目知识）构建块：** Loop Engineering 明确指出 *"A skill is how you stop re-explaining the same project context every session."* 将项目上下文写成 SKILL.md，agent 不再每次猜测。

**MemoryAgent — 知识外部化：** *"The agent forgets, the repo doesn't."* 项目的运行时知识应该持久化到磁盘，每次 agent 会话从中读取。

**Intent Debt（意图债务）：** 未外部化的意图（"这个项目怎么初始化"、"ctx 里该放什么"）在每次 agent 会话中都要重新付出代价。debugloop 的每一轮迭代都在重复支付这笔债务。

### 1.3 具体实施步骤

**Phase 1：自动提取 + 注入（P0，对应材料A Phase 1-3）**

```
1a. 新增 _build_project_context() 函数
    - 扫描 conftest.py，AST 解析提取 fixture 定义（函数名、返回值类型、依赖）
    - 扫描入口文件（run_server.py / app.py / main.py），提取初始化模式
    - 扫描 tests/ 目录，按文件名匹配已有测试范例
    - 扫描 README.md，提取 Quick Start 和架构说明段落
    - 扫描目录结构，生成分层描述

1b. 在 build_user_prompt() 中注入 project_context section
    - 位置：API manifest 之后，未覆盖分支之前
    - 格式：结构化的 Markdown，每个 section 带长度限制

1c. 对 pyOV 做回归测试，验证 handler 层测试通过率提升
```

**Phase 2：dl-map.md 人类覆盖机制（P1，对应材料A Phase 4）**

```
2a. 在 _build_project_context() 中增加 dl-map.md 检测
    - 文件存在 → 跳过自动提取，直接读取注入
    - 文件不存在 → 执行自动提取
2b. 输出提示：建议人类为复杂项目编写 dl-map.md
```

**Phase 3：长期 — 项目知识持久化为 Skills（P2）**

```
3a. 将自动提取的结果缓存到 .debugloop/skills/project-context.md
    - 首次提取后持久化，后续运行检测变更决定是否刷新
3b. 支持用户编辑缓存文件，人工补充自动提取未覆盖的知识
3c. 跨项目复用：不同项目生成各自的 project-context.md
```

### 1.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P0（Phase 1）/ P1（Phase 2）/ P2（Phase 3） |
| **预期收益** | handler 层测试通过率从 0/3 → ≥1/3（仅 Phase 1）；2-3 轮后 handler 覆盖率从 ~0% → 30-50% |
| **工作量** | Phase 1: 2-3 天 / Phase 2: 0.5 天 / Phase 3: 1 天 |
| **风险** | Prompt 长度膨胀（缓解：每 section 长度限制）；范例匹配错误（缓解：优先文件名前缀匹配，回退到内容匹配） |

---

## 建议 2：自动化运行 — 心跳机制

### 2.1 映射到 debugloop 具体问题

**现状：** debugloop 当前是手动触发的单次运行。用户执行 `./scripts/run.sh` → 跑 N 轮 → 产出报告 → 停止。每次需要人工介入启动。

**问题：**
- 没有持续监控覆盖率变化的能力
- 用户必须记住定期运行
- 无法在夜间或非工作时间自动迭代
- 覆盖率的退化（新代码引入后旧测试不再覆盖）无法自动检测

### 2.2 引用 Loop Engineering 原则

**Automations（心跳）构建块：** *"Automations are what make a loop an actual loop and not just one run you did once. They are the heartbeat: a recurring trigger that surfaces work without you asking."*

**LoopController — 闭环调度：** Loop Engineering 的 LoopController 负责"协调所有 agent 的运行节奏"和"判断当前循环是否完成"。debugloop 缺少这个调度层。

**Ralph 技法：** *"The while loop becomes a scheduled automation."* 最初的 Ralph 循环就是 `while ! grep -q "ALL TASKS DONE" STATUS.md; do ... done`。debugloop 需要将单次运行变成定时触发。

### 2.3 具体实施步骤

```
1. 新增定时调度模块（P1）
    - 支持 cron 表达式配置运行频率（如每日 2:00 AM）
    - 支持事件触发：git push → 自动运行 debugloop
    - 配置文件：.debugloop/schedule.yaml
      schedule: "0 2 * * *"
      trigger_on_push: true
      projects: ["pyOV", "project-b"]

2. 增量模式（P1）
    - 检测自上次运行以来的代码变更（git diff）
    - 仅对变更文件的未覆盖分支生成测试
    - 大幅减少每次运行的 token 消耗

3. 守护模式（P2）
    - 持续运行直到覆盖率目标达成或预算耗尽
    - 类似 /goal 命令：指定目标覆盖率 + 最大轮次/预算
    - 每轮间有可控的冷却间隔
```

### 2.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P1 |
| **预期收益** | 覆盖率提升从"手动触发"变为"自动持续"；夜间可多跑 3-5 轮，累积覆盖率提升 10-20% |
| **工作量** | 1-2 天 |
| **风险** | 无人值守时 token 消耗失控（缓解：预算上限 + 轮次上限）；夜间运行产生的问题无人及时发现（缓解：失败通知机制） |

---

## 建议 3：质量门禁 — 对抗验证引入

### 3.1 映射到 debugloop 具体问题

**核心问题（材料A §1.3）：** debugloop 当前的测试验证能力不足。3 个生成的测试全部失败，但系统将失败类型笼统归类为"AssertionError / 类型错误 / 未归类错误"，没有区分"测试本身写错了"（setup 崩溃）vs "被测代码有 bug"（真正的断言失败）。

**更具体的验证缺陷：**
- debugloop 对 3 个失败测试都判定为"源码问题"而非"测试生成问题"，但实际上日志没有显示测试通过了 ctx 构造——它们很可能在 setup 阶段就抛异常了
- 没有验证"生成的测试是否能正确调用被测函数"这一步
- 测试质量（断言是否有效、是否覆盖目标分支）未经验证
- 没有独立的验证视角——写测试的 LLM 也负责判断测试好不好

### 3.2 引用 Loop Engineering 原则

**Sub-agents（maker/checker 分离）：** *"One agent writes the code, a different one reviews it."* Loop Engineering 的核心原则：写代码的 agent 和检查代码的 agent 必须分离。

**CriticAgent — 对抗验证：** *"This is the core differentiator from ordinary multi-agent orchestration. CriticAgent doesn't 'review' — it tries to overturn."* CriticAgent 用 3 个独立视角评估：
- **Correctness（正确性）：** 测试是否正确调用了被测函数？输入数据是否合法？
- **Completeness（完整性）：** 测试是否真正覆盖了目标分支？断言是否验证了关键输出？
- **Robustness（健壮性）：** 测试是否能在不同环境下运行？是否依赖外部状态？

**2/3 通过才算通过：** *"Default: reject uncertain results."*

**Boris Cherny 的观点：** *"The model that wrote code is way too nice grading its own homework."*

### 3.3 具体实施步骤

```
1. 分离"测试生成"和"测试验证"为两个独立 Agent（P0）
    - ExecutorAgent（现有 LLM）：负责生成测试代码
    - CriticAgent（新增 LLM 调用）：负责验证生成的测试
    - 两个 Agent 使用不同 prompt、不同视角
    - 可选：使用不同模型（如一个用 Claude、一个用 GPT）

2. 实现 3 视角对抗验证（P0）
    a. Correctness 视角（P0 最高优先级）
       - 静态分析：测试代码语法是否正确？import 是否存在？
       - 运行验证：测试能否成功执行到 setUp 完成？
       - 目标验证：测试是否真正调用了目标未覆盖分支所在的函数？
       - 失败分类：区分 "setup 崩溃"（测试生成问题） vs "断言失败"（可能是被测代码问题）

    b. Completeness 视角（P1）
       - 分支追踪：验证生成的测试执行路径是否包含目标未覆盖分支
       - 断言有效性：断言是否验证了关键输出值（而非仅 assert True）
       - 边界覆盖：异常路径和边界条件是否被考虑

    c. Robustness 视角（P2）
       - 环境无关性：测试是否依赖特定的文件路径/环境变量
       - Mock 合理性：Mock 策略是否合理（不过度 Mock 也不缺少 Mock）
       - 隔离性：测试之间是否相互独立

3. 门禁决策（P0）
    - Correctness 必须通过（setup 不崩溃 + 调用目标函数）
    - Correctness + Completeness 通过 → 测试被接受
    - 任一失败 → 生成反馈信息，反馈给 ExecutorAgent 重新生成
    - 反馈必须具体：不是"测试不好"，而是"第 15 行 ctx['store'] 缺少 TripleStore 初始化"

4. 反馈闭环（P0）
    - 对抗验证的失败信息 → 格式化为结构化反馈 → 注入下一轮 ExecutorAgent 的 prompt
    - 让 ExecutorAgent 从上一轮的错误中学习
    - 记录到 MemoryAgent（见建议 4）
```

### 3.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P0（Correctness 视角 + 门禁）/ P1（Completeness）/ P2（Robustness） |
| **预期收益** | 将"setup 崩溃"和"真正断言失败"区分开，减少无效迭代；Correctness 门禁可将测试通过率从当前 0/3 提升到 ≥2/3（单轮）；反馈闭环使后续轮次逐步改进 |
| **工作量** | P0: 2-3 天 / P1: 1-2 天 / P2: 1 天 |
| **风险** | CriticAgent 的 LLM 调用增加 token 消耗（每次验证约 ¥0.5-1）；CriticAgent 本身可能误判（缓解：Correctness 用确定性检查 + LLM 双重验证） |

---

## 建议 4：记忆与反馈闭环

### 4.1 映射到 debugloop 具体问题

**问题（材料A §1.1-1.3）：** debugloop 的每轮迭代是独立的——第 2 轮不知道第 1 轮为什么失败，第 3 轮不知道前两轮试了什么。

**具体表现：**
- 第 1 轮：生成测试 1（ctx 构造错误 → 崩溃）。debugloop 判定"源码问题"，继续。
- 第 2 轮：生成测试 2（同样的 ctx 构造错误 → 崩溃）。debugloop 没有识别出这是同样的问题。
- 第 3 轮：生成测试 3（还是 ctx 构造错误 → 崩溃）。
- 2 轮后覆盖率零提升 → stagnation 终止。

**如果有了记忆：** 第 1 轮失败后，系统记录"所有测试在 ctx 构造阶段崩溃，ctx 需要包含 store/tbox/abox/mapper"。第 2 轮的 prompt 包含这段记忆，LLM 不再犯同样的错误。

### 4.2 引用 Loop Engineering 原则

**Memory（外部记忆）构建块：** *"The agent forgets, the repo doesn't. The state has to live on disk, not in the context window."*

**MemoryAgent：** Loop Engineering 的 MemoryAgent 负责 *"跨会话持久化知识，为 Planner 提供上下文"*，记录 *"什么有效、什么失败、为什么"*，而不仅仅是结果。

**Intent Debt：** Addy Osmani 提出的概念——未外部化的意图在每次会话中都要重新付出代价。debugloop 的每一轮都在支付"ctx 怎么构造"这笔意图债务。

**Ralph 技法：** *"The intelligence lives in verifiable outcomes, applied over and over against an external memory the model cannot pollute."* 记忆在磁盘上，不在 LLM 的上下文窗口里。

### 4.3 具体实施步骤

```
1. 建立失败模式知识库（P0）
    - 存储位置：.debugloop/memory/failure-patterns.json
    - 每条记录包含：
      - 失败类型（setup_crash / assertion_failure / import_error / ...）
      - 失败文件 + 行号
      - 错误信息摘要
      - 根因分析（由 CriticAgent 或 LLM 生成）
      - 修复建议
    - 去重：相同/相似的失败不重复记录

2. 每轮 prompt 注入历史失败摘要（P0）
    - 在 build_user_prompt() 中新增 "## 历史失败记录" section
    - 格式：
      上一轮失败：
      - test_handle_tree_1.py:15 — ctx['store'] 未初始化
        → 根因：需要 TripleStore().load_directory(onto_dir)
        → 建议：参考 tests/test_handlers.py 的 setUp 方法
    - 限制：最多注入最近 5 条失败记录

3. 成功模式记忆（P1）
    - 记录成功生成的测试的特征：
      - 使用了哪个 fixture / 初始化模式
      - 采用了哪种测试风格（HTTP 请求 vs 直接调用）
      - 什么样的断言结构被 CriticAgent 判定为有效
    - 在 prompt 中注入"成功模式"作为正例

4. 覆盖率演变追踪（P2）
    - 存储位置：.debugloop/memory/coverage-history.json
    - 记录每轮的覆盖率快照 + 成功/失败测试数
    - 支持趋势分析：哪些模块覆盖率在上升、哪些停滞、哪些退化
    - 为 PlannerAgent 提供数据，指导下一轮的目标分支选择

5. Stagnation 检测升级（P0）
    - 当前：2 轮覆盖率零提升 → 终止
    - 升级：区分"测试生成失败导致零提升" vs "测试通过但覆盖不到" vs "分支确实难以覆盖"
    - "测试生成失败" → 不应终止，应走反馈闭环重新生成
    - "测试通过但覆盖不到" → 记录到记忆，请求 LLM 调整策略
    - "分支确实难以覆盖" → 标记为需要人工介入
```

### 4.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P0（失败模式知识库 + 历史注入 + Stagnation 升级） |
| **预期收益** | 避免"重复犯同样的错误"——同一类 ctx 构造错误不会在第 2、3 轮重现；Stagnation 检测升级后，debugloop 不再因"测试生成失败"而误判为覆盖停滞 |
| **工作量** | P0: 2 天 / P1: 1 天 / P2: 1 天 |
| **风险** | 记忆膨胀——失败记录越来越多（缓解：去重 + 最近 N 条限制）；错误记忆污染——如果根因分析错误，误导后续轮次（缓解：CriticAgent 的 Correctness 验证兜底） |

---

## 建议 5：Worktree 隔离 — 并行测试生成

### 5.1 映射到 debugloop 具体问题

**现状：** debugloop 是串行执行的——每轮生成测试 → 运行 → 收集覆盖率 → 下一轮。一次只处理一个未覆盖分支。

**潜在问题：**
- 不同模块的 handler 之间没有依赖关系，可以并行生成测试
- pyOV 有 10 个 handler 文件、约 942 个未覆盖分支，串行处理效率极低
- 不同 LLM 调用生成的测试可能写入同一个测试文件，产生冲突
- 失败的测试可能污染运行环境，影响后续测试的覆盖率收集

### 5.2 引用 Loop Engineering 原则

**Worktrees（工作树隔离）：** *"The moment you run more than one agent, files start colliding. A git worktree solves it: a separate working directory on its own branch."*

**并行执行：** Loop Engineering 的 ExecutorAgent 支持多个 Agent 并行执行独立任务。TaskQueue + DAG 确保依赖满足后才执行。

**Ralph 技法：** 每次迭代 fresh context + 独立的 checkout，不同任务之间不互相污染。

### 5.3 具体实施步骤

```
1. 并行测试生成（P2）
    - 识别无依赖关系的未覆盖分支（不同模块、不同文件）
    - 为每个独立模块创建独立的 LLM 会话
    - 生成的测试写入不同的临时文件（如 test_handlers_ontology_gen.py）
    - 所有模块完成后，合并测试文件

2. 测试环境隔离（P2）
    - 每轮生成的测试写入独立的临时目录
    - 运行前验证：所有生成的测试编译/语法正确
    - 失败的测试不影响其他测试的覆盖率收集
    - 通过测试 → 合并到正式测试文件；失败测试 → 丢弃并记录

3. Git worktree 集成（P2 — 长期）
    - 为每个并行任务创建独立 worktree
    - 任务完成 → 合并成功分支 → 清理 worktree
    - 适合大规模重构或跨多模块的覆盖率提升
```

### 5.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P2 |
| **预期收益** | 10 个 handler 模块可并行处理，运行时间缩短至 1/3-1/5；减少测试间污染 |
| **工作量** | 2-3 天 |
| **风险** | 并行 LLM 调用增加并发 token 消耗；合并冲突需要处理策略（缓解：写入独立文件 + 顺序合并） |

---

## 建议 6：验证化的停止条件

### 6.1 映射到 debugloop 具体问题

**问题（材料A §1.1-1.3）：** debugloop 当前的停止条件主要是"覆盖率停滞"（2 轮零提升 + 总覆盖率无增长）。这是被动式的止损条件，不是主动式的目标达成条件。

**具体表现：**
- pyOV 运行中，2 轮后覆盖率从 33.9% 没变 → stagnation 终止
- 但没有区分"测试写不出来" vs "测试写出来了但覆盖不到" vs "分支确实无法覆盖"
- 用户设置的目标覆盖率 85% 只是一个数字，没有变成实际的停止条件判断逻辑

**Loop Engineering 式的停止条件应该长什么样：**
- "pyOV handler 层覆盖率 ≥ 50%"
- "且所有生成的测试通过 Correctness 门禁"
- "且最近 3 轮有正向覆盖率提升"
- "或总轮次达到 20 轮"
- "或总 token 消耗超过 ¥50"

### 6.2 引用 Loop Engineering 原则

**可验证的停止条件：** Loop Engineering 强调 *"Write the stop condition like a contract, not a wish."*

| 弱的版本 | 可验证的版本 |
|----------|-------------|
| "提升测试覆盖率" | "src/api/handlers/ 覆盖率 ≥ 50%" |
| "感觉差不多了" | "npm test exits 0 且 coverage report 确认数字" |
| 无约束 | "最多 20 轮或 ¥50" |

**Ralph 技法：** *"The 'ALL TASKS DONE' check becomes a /goal condition graded by a separate model."*

**Termination conditions：** Loop Engineering Core 定义了三种终止条件：
- `goal_successfully_completed` — CriticAgent 确认所有成功标准满足
- `max_loop_iterations_reached` — 防止无限循环
- `critical_failure_detected` — 不可恢复的错误

### 6.3 具体实施步骤

```
1. 支持分层覆盖率目标（P1）
    配置格式（.debugloop/config.yaml）：
    targets:
      overall: 85%
      layers:
        - path: "src/api/handlers/"
          target: 50%
          priority: high
        - path: "src/engine/"
          target: 70%
          priority: medium

2. 多维停止条件（P1）
    a. 正向条件（全部满足 → 成功终止）
       - 分层覆盖率目标全部达成
       - 最新生成的测试全部通过 CriticAgent 的 Correctness 门禁

    b. 保护条件（任一触发 → 安全终止）
       - 总轮次达到上限（默认 20 轮）
       - 总 token 消耗达到预算上限（默认 ¥50）
       - 连续 3 轮无新测试通过 Correctness 门禁

    c. 异常条件（触发 → 告警终止）
       - 测试运行环境崩溃（pytest 无法启动）
       - 覆盖率收集工具异常（coverage 报告解析失败）

3. 停止原因可审计（P1）
    - 终止时输出清晰的停止原因报告：
      停止原因：目标达成 — handler 层覆盖率 52%（目标 50%）✓
      总轮次：7 / 20
      总消耗：¥18.42 / ¥50
      成功测试：12 个
      失败测试：5 个
    - 失败终止时输出"最接近目标"的差距分析

4. /goal 命令支持（P2）
    - 类似 Claude Code 的 /goal 命令
    - debugloop /goal "pyOV src/api/handlers 覆盖率达到 60%"
    - 自动运行直到目标达成或预算耗尽
```

### 6.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P1 |
| **预期收益** | 停止条件从"被动的放弃"变为"主动的达成"；保护条件防止 token 浪费；分层目标使覆盖率的提升更有针对性 |
| **工作量** | 1 天 |
| **风险** | 目标设置过高导致永远跑不到（缓解：保护条件兜底） |

---

## 建议 7：Ralph 技法 — 新鲜上下文 + 原子化工作单元

### 7.1 映射到 debugloop 具体问题

**隐性但重要的问题：** debugloop 当前的 prompt 构造策略是每轮重新构造，但 LLM 会话可能复用（取决于实现）。如果复用了长会话，上下文窗口中的旧推理、死胡同、过时的文件内容会逐渐污染决策质量。

**Ralph 技法的洞察：** *"The context reset is key. A long agent session degrades as the window fills with old reasoning, dead ends, and stale file contents."* debugloop 应该确保每轮 LLM 调用都从一个干净的上下文开始。

### 7.2 引用 Loop Engineering 原则

**Ralph 技法核心原则：**
1. Fresh context: 每轮新 LLM 实例
2. Read current repo state: 从磁盘读取当前状态，不依赖记忆
3. Do one unit of work: 每轮只做一个原子任务
4. Commit on success: 成功的工作立即持久化
5. Exit: 完成后退出，不滞留

### 7.3 具体实施步骤

```
1. 确保每轮 LLM 会话独立（P0）
    - 每轮测试生成启动全新的 LLM 会话
    - 不依赖前一轮的对话历史
    - 所有上下文从结构化 prompt + 磁盘状态中获取

2. 原子化任务定义（P1）
    - 每轮只处理一个未覆盖分支（或一个紧密相关的分支组）
    - 生成测试 → 运行 → 验证 → 通过则持久化 → 进入下一轮
    - 不一次生成多个不相关模块的测试

3. 状态文件驱动（P1）
    - 维护 .debugloop/state.json 记录：
      当前轮次、剩余未覆盖分支列表、已覆盖分支列表、最近成功率
    - 每轮开始前读取状态文件
    - 每轮结束后更新状态文件
    - LLM 不维护状态——状态在磁盘上
```

### 7.4 优先级和预期收益

| 指标 | 数值 |
|------|------|
| **优先级** | P0（独立会话）/ P1（原子化 + 状态文件） |
| **预期收益** | 消除长会话的上下文污染；原子化让每轮目标更明确，提高单轮成功率 |
| **工作量** | 0.5 天（独立会话）+ 1 天（原子化 + 状态文件） |
| **风险** | 每次 fresh context 意味着每次重新发送 prompt（token 消耗增加）；状态文件的读写需要原子化防竞态（单进程串行执行不需要） |

---

## 综合优先级路线图

```
Week 1 (P0 — 立即执行，最大收益)
├── 建议 1-Phase1：项目知识自动提取+注入（fixture/范例/README）
├── 建议 3-Phase1：对抗验证 Correctness 视角 + maker/checker 分离
├── 建议 4-Phase1：失败模式知识库 + 历史注入 + Stagnation 升级
└── 建议 7-Phase1：确保每轮 LLM 会话独立

Week 2 (P1 — 核心增强)
├── 建议 2：定时调度 + 增量模式
├── 建议 3-Phase2：Completeness 视角对抗验证
├── 建议 4-Phase2：成功模式记忆
├── 建议 6：分层覆盖率目标 + 多维停止条件
└── 建议 7-Phase2：原子化任务 + 状态文件驱动

Week 3+ (P2 — 锦上添花)
├── 建议 1-Phase3：项目知识持久化为 Skills 文件
├── 建议 3-Phase3：Robustness 视角对抗验证
├── 建议 4-Phase3：覆盖率演变追踪
├── 建议 5：并行测试生成 + Worktree 隔离
└── 建议 1-Phase2：dl-map.md 人类覆盖机制
```

---

## 预期综合收益

以 pyOV 项目为基线，各建议叠加后的预期效果：

| 阶段 | 累积改进 | handler 通过率 | handler 覆盖率 | 关键变化 |
|------|---------|---------------|---------------|---------|
| 当前 | — | 0/3 (0%) | ~0% | 3 个测试全部 setup 崩溃 |
| Week 1 | 建议1+3+4+7 | ≥1/3 (33%) | 10-20% | LLM 知道 ctx 结构 + setup 崩溃被识别 + 不再重复错误 |
| Week 2 | +建议2+6 | 2-3/轮 | 30-50% | 自动定时运行 + 分层目标驱动 + 多维停止 |
| Week 3+ | +建议5 | 3-5/轮 | 50-70% | 并行处理多模块 + 完整对抗验证 |

**保守估计：3 周后 handler 层覆盖率可从接近 0% 提升到 50-70%，总体覆盖率从 33.9% 提升到 60-75%。**

---

## 附录：debugloop 问题 — Loop Engineering 原则映射速查表

| debugloop 问题 | 严重程度 | Loop Engineering 原则 | 建议编号 | 优先级 |
|---------------|---------|----------------------|---------|--------|
| LLM 不知道 ctx 结构 | 🔴 致命 | Skills — 项目知识注入 | 建议 1 | P0 |
| 测试范例未被利用 | 🔴 致命 | Skills — 项目知识注入 | 建议 1 | P0 |
| 测试失败原因未分类 | 🔴 致命 | CriticAgent — 对抗验证 | 建议 3 | P0 |
| 每轮不吸取教训 | 🔴 致命 | MemoryAgent — 外部记忆 | 建议 4 | P0 |
| 缺乏独立验证 | 🟡 严重 | Sub-agents — maker/checker 分离 | 建议 3 | P0 |
| Stagnation 检测粗糙 | 🟡 严重 | 可验证停止条件 | 建议 4+6 | P0/P1 |
| 手动触发运行 | 🟡 严重 | Automations — 心跳机制 | 建议 2 | P1 |
| 停止条件模糊 | 🟡 严重 | 验证化的停止条件 | 建议 6 | P1 |
| 串行处理效率低 | 🟢 改进 | Worktrees — 并行隔离 | 建议 5 | P2 |
| 项目知识未持久化 | 🟢 改进 | Skills — 持久化项目上下文 | 建议 1-Phase3 | P2 |
| 可能复用长会话 | 🟡 严重 | Ralph 技法 — fresh context | 建议 7 | P0 |

---

> 编制依据：
> - debugloop-context-injection-analysis.md（pyOV 实战分析，2026-06-27）
> - LoopEngineering-Core v1.0 系统规范
> - Loop Engineering Guide (lushbinary.com)
> - Addy Osmani / Boris Cherny / Peter Steinberger 的 Loop Engineering 论述
