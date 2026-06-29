# 立项书静态审查报告 — team-loop v1.41

> 审查时间: 2026-06-28 16:41 CST
> 审查文件: `projects/team-loop/team-loop.md` (~2100 行, v1.41)
> 审查维度: 逻辑一致性 / 完整性 / 可执行性 / 风险点
> 审查人: reviewer (zai/glm-5.2)

---

## 审查概况

- **问题统计**: Critical: 5 / High: 9 / Medium: 12 / Low: 5
- **审查结论**: 🔴 **需修复后重审** — 存在 5 个 Critical 级问题，涉及核心流程结构断裂、角色输入契约矛盾、并行设计不可行等，必须修复后方可进入编码阶段。

---

## 一、逻辑一致性

### 🔴 Critical

**[BLOCK-C1] §3.8.4 seq 9（记忆沉淀）完全孤立，无任何 flow 连接**

- **位置:** §3.8.4 flows 定义 (约第 1870-1890 行)
- **问题:** 12 条 flow (f1-f12) 中没有任何一条连接到 seq 9。seq 9 的 publicist cell 声明 `in: ["TaskResult", "Decision"]`，但没有 flow 向 seq 9 传递数据；`out: ["MemoryEntry"]` 也没有 flow 消费。seq 9 是流程中的孤岛。
- **风险:** 记忆闭环是 §2.2 原则 5 的核心设计，如果 seq 9 不可达，publicist 永远不会执行，Memory 永远不会写入，PlannerAgent 下一轮无法查询历史——整个学习闭环断裂。
- **修复建议:**
  - 增加 f13: `main seq7 → publicist seq9` (async, what: "Decision + TaskResult")
  - 增加 f14: `publicist seq9 → main seq2` (async, what: "MemoryEntry")，使下轮 PlannerAgent 可查询
  - 或者将 seq 9 移至 seq 7 之后、seq 8 之前，与 §2.5 伪代码顺序一致

**[BLOCK-C2] §2.5 契约伪代码与 §3.8.4 流程在结构上不对应**

- **位置:** §2.5.1 (约第 80-110 行) vs §3.8.4 (约第 1810-1890 行)
- **问题:** §2.5 的结构是 `while → planner → for task in KRs → [coder → reviewer → tester → auditor → publicist] → check FATAL → done`。关键点：
  1. §2.5 有 `for task in 就绪的KR` 循环（一次处理多个 KR），§3.8.4 描述"一次KR执行的标准流程"（只处理一个 KR），两者粒度不一致。
  2. §2.5 中 publicist 在每个 KR 的 if/else 中被调用（per-KR 记忆），§3.8.4 中 seq 9 在 seq 8（终止判断）之后——意味着记忆在终止判断之后才记录。
  3. §2.5 中 FATAL 检查在 `for` 循环结束后（一轮所有 KR 执行完后），§3.8.4 的 seq 8 在单个 KR 执行后就做终止判断。
- **风险:** 开发者按 §2.5 理解还是按 §3.8.4 实现，会得到完全不同的系统行为。
- **修复建议:** 统一为一套描述。建议以 §3.8.4 为权威，修正 §2.5 伪代码使其与 §3.8.4 一致；或在 §3.8.4 中明确标注 `for` 循环的嵌套关系。

**[BLOCK-C3] §3.4 check_order 遗漏两条终止规则**

- **位置:** §3.4 (约第 270-310 行)
- **问题:** `check_order` 数组列出 5 条：`["all_kr_completed", "basic_layer_fatal", "budget_exhausted", "max_rounds", "stagnation"]`，但 `rules` 数组定义了 7 条（多出 `ks_unreachable` 和 `agent_timeout`）。这两个额外的 FATAL 终止规则不在 check_order 中，意味着它们永远不会被检查。
- **风险:** KS 不可达或 agent 超时时，LoopController 不会终止，loop 会无限运行。
- **修复建议:** 将 `ks_unreachable` 和 `agent_timeout` 加入 check_order，明确其优先级位置。

**[BLOCK-C4] §3.4 终止条件与 §2.5 while 循环的 STATE.md 机制脱节**

- **位置:** §2.5.1 (约第 82 行) vs §3.4 (约第 270-330 行)
- **问题:** §2.5 的 while 循环条件是 `grep -q "终止" .team-loop/STATE.md`。§3.4 定义了 7 条终止规则和对应的 result（SUCCESS/FATAL/BUDGET/MAX_ROUNDS/STAGNATION），但**从未说明谁、在什么时候、以什么格式写入 STATE.md**。§2.5 只展示了 `echo "FATAL" >> STATE.md` 这一种情况。其他 6 种终止结果如何写入 STATE.md？是 LoopController 写的吗？
- **风险:** 如果 STATE.md 未被正确写入，while 循环永远不会退出。
- **修复建议:** 在 §3.4 终止后行为中明确：LoopController 在判定终止后，将 result 字符串写入 `.team-loop/STATE.md`，格式规范如 `TERMINATED: <result>\nREASON: <rule_id>\nTIMESTAMP: <iso8601>`。

### 🟠 High

**[MUST-FIX-H1] §3.6.5 引用的 seq 编号与 §3.8.4 不匹配**

- **位置:** §3.6.5 (约第 670 行)
- **问题:** §3.6.5 称"详见 §3.8.4 seq 4-6 的三视角并行验证流程（seq 4 = 三视角并行验证，seq 5 = 2/3裁决，seq 6 = 终止判断）"。但 §3.8.4 中 seq 4 = coder 执行，seq 5 = main 分发，seq 6 = 三视角并行，seq 7 = 2/3裁决，seq 8 = 终止判断。所有引用偏移了 2。
- **修复建议:** 将 §3.6.5 的引用修正为 "seq 6-8"。

**[MUST-FIX-H2] §2.5 的 FATAL 检查不完整**

- **位置:** §2.5.1 (约第 108-110 行)
- **问题:** §2.5 只展示了 `if [ K1基础层 连续3轮全失败 ]; then echo "FATAL" >> STATE.md`，但 §3.4 定义了 3 种 FATAL（basic_layer_fatal / ks_unreachable / agent_timeout）。另外 SUCCESS / MAX_ROUNDS / BUDGET / STAGNATION 也需要写入 STATE.md，但 §2.5 未展示。
- **修复建议:** 修正 §2.5 伪代码，将 FATAL 检查替换为通用终止条件检查，或在伪代码注释中声明"简化表示，完整规则见 §3.4"。

**[MUST-FIX-H3] §3.8.4 f12 违反 I4 输入约束**

- **位置:** §3.8.4 flows f12 (约第 1890 行) vs §3.8.3.1 I4 约束
- **问题:** f12 从 `main seq8` 传递 `Decision` 到 `main seq2`。但 seq2 的 `in: ["StandardGoal"]` 不包含 `"Decision"`。根据 §3.8.3.1 的 I4 约束："to.seq 对应 cell 的 in 数组包含 what → 否则拒绝发送，WARN"。f12 会每次触发 WARN。
- **修复建议:** 在 seq2 的 `in` 中增加 `"Decision"` 和 `"MemoryEntry"`，反映反馈环带来的额外输入。

---

## 二、完整性

### 🔴 Critical

**[BLOCK-C5] auditor 并行执行与其输入契约根本矛盾**

- **位置:** §3.1.2 (约第 190 行) + §3.6.6 (约第 655 行) + §3.8.4 seq 6 (约第 1850 行)
- **问题:** 三处定义互相矛盾：
  1. §3.1.2: auditor 输入 = `TaskResult + 审查记录`
  2. §3.6.6: auditor checklist 包含 "reviewer审查完整性" 和 "tester隔离完整性" — 需要看到 reviewer 和 tester 的 Verdict
  3. §3.8.4 seq 6: reviewer/tester/auditor **并行**执行，auditor 的 `in: ["TaskResult"]` — 没有 reviewer/tester 的 Verdict

  auditor 被设计为"审计审查者有没有审查到位"（§3.6.1），但在 seq 6 中与被审计者同时执行，根本无法获取它们的产出。这是设计层面的逻辑不可能。
- **风险:** auditor 的 12 项 checklist 中至少 4 项无法执行（reviewer 审查完整性、tester 隔离完整性、CR 记录完整性、流程合规），2/3 投票机制退化为 2 视角（reviewer + tester），robustness 维度形同虚设。
- **修复建议（三选一）:**
  1. **拆分 auditor 为两阶段:** seq 6a = reviewer + tester 并行 → seq 6b = auditor 审查（输入含 TaskResult + reviewer.Verdict + tester.Verdict + tester.CoverageReport）
  2. **auditor 改为串行第三步:** seq 6 = reviewer + tester 并行 → seq 6.5 = auditor
  3. **重新定义 auditor 职责:** 如果坚持并行，则 auditor 只审 TaskResult 本身（不审 reviewer/tester 的审查质量），需修改 §3.6.6 checklist

### 🟠 High

**[MUST-FIX-H4] MonitorAgent 在 §3.8.4 流程中缺失**

- **位置:** §3.1.1 (约第 165 行) + §3.1.2 (约第 195 行) + §3.8.4
- **问题:** §3.1.1 定义 MonitorAgent（由 main 承担）产出 `RoundSummary`，§3.1.2 定义其输出。但 §3.8.4 的 9 个 seq 中没有任何一个包含 MonitorAgent 的任务。`RoundSummary` 也不在 7 种交接物之外（§3.7.4 定义了 H7 = RoundSummary，但没有 flow 传递它）。
- **修复建议:** 在 §3.8.4 中增加 MonitorAgent 步骤（如 seq 8.5 或合入 seq 8），增加对应 flow。

**[MUST-FIX-H5] CoverageReport 缺少正式 Schema**

- **位置:** §3.7.4 (约第 1090-1190 行) + §3.8.4 seq 6 tester out
- **问题:** §3.8.4 seq 6 中 tester 的 `out: ["Verdict", "CoverageReport"]`，§3.1.2 也定义 tester 输出含 `CoverageReport`。但 §3.7.4 的 7 种交接物（H1-H7）中没有 CoverageReport 的 Pydantic 定义。它应该包含什么字段？file/baseline_coverage/new_coverage/delta？仅在 §3.7.4.1 tester 模板中以注释形式提到。
- **修复建议:** 在 §3.7.4 中增加 H8: CoverageReport 的正式 Pydantic Schema。

**[MUST-FIX-H6] Task 和 TaskContext 缺少正式 Pydantic 定义**

- **位置:** §3.7.4 (约第 1130-1150 行)
- **问题:** `TaskDAG` (H2) 内嵌 `Task` 类型，`TaskUnit` (H3) 内嵌 `TaskContext` 类型，但两者都只以注释形式出现（`# Task: {task_id, description, ...}`），没有正式的 Pydantic BaseModel 定义。`verification` 字段在注释中提及但不在正式 Schema 中。这对编码来说不够——开发者需要知道每个字段的类型、是否必填、约束等。
- **修复建议:** 为 Task 和 TaskContext 增加独立的 Pydantic BaseModel 定义，包含所有字段类型和验证规则。

**[MUST-FIX-H7] KR 状态机未定义**

- **位置:** §3.8.4 全文
- **问题:** Task 有 `status` 字段，但合法状态值和转换规则从未定义。pending → in_progress → completed/failed/skipped？retried 算什么状态？什么是"就绪的KR"（§2.5 `for task in 就绪的KR`）——ready 状态？谁来设置状态？
- **修复建议:** 定义 Task 状态枚举和状态转换图，明确每个状态的进入/退出条件。

**[MUST-FIX-H8] "可重试"判定规则未定义**

- **位置:** §3.8.4 seq 7 (约第 1870 行)
- **问题:** seq 7 描述"失败可重试则回到seq2重新规划，失败不可重试则跳过此KR"。但什么情况"可重试"、什么情况"不可重试"？没有定义规则。retry_count 上限是多少？§3.4 提到 `retry_count >= max_retries` 用于 agent_timeout 终止，但 max_retries 的值未定义。
- **修复建议:** 定义重试判定规则，如：可重试条件（非 FATAL 错误 + retry_count < max_retries）、max_retries 默认值（如 2）、不可重试条件（FATAL 错误、依赖的 KR 已失败等）。

### 🟡 Medium

**[SHOULD-FIX-M1] main 的四重角色缺乏运行时隔离机制**

- **位置:** §3.1.1 (约第 165 行) + §3.1.2
- **问题:** main 同时承担 GoalAgent / PlannerAgent / LoopController / MonitorAgent 四个角色。§3.1.2 对每个角色有不同的工具约束（如 GoalAgent "不查询项目知识，不访问 KS"，PlannerAgent "必须查询 KS"）。但在 §3.8.4 中它们都是 "main" 列，没有说明如何在运行时区分这些角色和强制工具约束。
- **建议:** 说明角色切换机制（如 main 内部的函数分离、prompt 切换等）和约束强制方式。

**[SHOULD-FIX-M2] tester 工具约束 "write (源文件)" 语义模糊**

- **位置:** §3.1.2 (约第 190 行)
- **问题:** tester 不可用工具标注为 "write (源文件)"。test 文件算不算"源文件"？tester 需要写 coverage 配置、临时 fixture 吗？如果严格禁止 write，tester 如何搭建隔离环境？
- **建议:** 明确区分 "源码文件"（src/）和 "测试产物"（临时文件、coverage 配置），tester 可写后者不可写前者。

**[SHOULD-FIX-M3] §3.8.4 seq 5 输出数量与下游 flow 不匹配**

- **位置:** §3.8.4 seq 5 (约第 1840 行)
- **问题:** seq 5 main 的 `out: ["TaskResult","TaskResult","TaskResult"]`——同一类型出现 3 次。但 §3.8.3 字段约束说 `out` 是 `string[]`。同一类型重复在数组中，下游 f5/f6/f7 如何区分哪个是给谁的？这违反了 I3 约束的精确匹配。
- **建议:** 用语义化的输出名区分，如 `["TaskResult(review)", "TaskResult(test)", "TaskResult(audit)"]`，或改为单输出 `["TaskResult"]` 加上描述"fan-out 到 3 个 critic"。

**[SHOULD-FIX-M4] Budget 估算机制未定义**

- **位置:** §3.4 budget_exhausted 规则 (约第 290 行)
- **问题:** 条件 `tokens_consumed + estimated_next_round_tokens > budget_limit` 中的 `estimated_next_round_tokens` 如何计算？取上一轮消耗？取历史平均？取最大值？估算偏差大可能导致过早终止或超支。
- **建议:** 定义估算算法，如 `max(历史轮次均值 × 1.5, 上一轮消耗)`，并设置最低保留量。

**[SHOULD-FIX-M5] 记忆检索机制未定义**

- **位置:** §3.5.6 PlannerAgent prompt "先查询Memory" + §3.8.4 seq 2
- **问题:** PlannerAgent "查询Memory" 的具体方式未定义。是 SPARQL 查询 rdf/ast/memory.ttl？是直接读文件？是向量检索？查询性能如何保证？Memory 膨胀后如何处理？
- **建议:** 明确记忆存储格式（§3.7.4 H6 提到 rdf/ast/memory.ttl 但未展开）、查询接口、结果排序和数量限制。

**[SHOULD-FIX-M6] 全局知识库的写入和更新机制未定义**

- **位置:** 附录 D (约第 2090 行)
- **问题:** 定义了 `~/.openclaw/kb/team-loop/` 下的 patterns / llm-behaviors / anti-patterns 三个目录，但未说明谁写入这些全局知识、何时写入、格式是什么、如何与项目本地 Memory 区分。
- **建议:** 增加全局知识库的写入触发条件和格式定义，或标注为"v2 功能，v1 仅支持项目本地 Memory"。

---

## 三、可执行性

### 🟠 High

**[MUST-FIX-H9] Knowledge Server 实现状态未声明**

- **位置:** §3.2.1 (约第 220 行) + §4.1 (约第 1920 行)
- **问题:** §3.2.1 称"实现项目：projects/pyrdf-knowledge"，§4.1 标注 KS 可行性为"✅ 高"。但文档未说明 KS 是否已实现、是否有可用 API、还是需要从零开发。如果 KS 尚未实现，4-5 天的总工期不现实。
- **修复建议:** 在 §4.1 或 §3.2.1 中明确 KS 当前实现状态（已实现/原型/未开始），并据此调整工期。

### 🟡 Medium

**[SHOULD-FIX-M7] 工期估计过于激进**

- **位置:** §4.2 (约第 1940 行) + §5 (约第 2000 行)
- **问题:** 4-5 天要完成：6 个核心接口 + 7+ 个 Pydantic Schema + 三重输出日志框架 + DI 容器 + 6 个 mock 实现 + 6+ 个适配器 + 6 个 agent 核心逻辑 + CR 系统 + 端到端集成。新增代码量估计 ~1200 行，但日志框架+DI+CR+交接物校验+3个适配器+6个mock 就远超 1200 行。
- **建议:** 调整为 7-10 天，或明确标注"4-5 天为理想估计，不含调试和需求变更"。

**[SHOULD-FIX-M8] sessions_spawn 如何传递 TaskUnit 上下文未定义**

- **位置:** 附录 C.2 (约第 2050 行)
- **问题:** 示例只展示了 `task` 字符串和 `taskName`，但 TaskUnit 包含 TaskContext（goal, knowledge, past_failures, constraints）。这些结构化数据如何传递给子 agent？是序列化到 task 字符串中？还是用 attachments？还是写入文件让子 agent 读？
- **建议:** 增加完整的 sessions_spawn 调用示例，展示 TaskUnit 的序列化方式。

**[SHOULD-FIX-M9] 覆盖率测量工具链未指定**

- **位置:** §3.5.5 + §3.6.6 tester
- **问题:** 文档提到"分支覆盖率"和"pytest + coverage"，但未指定：使用 coverage.py 还是 pytest-cov？基线覆盖率如何建立和存储？增量覆盖率如何计算（对比哪个版本）？目标文件如何确定？
- **建议:** 增加"覆盖率测量规格"小节，明确工具、命令、报告格式、基线管理。

**[SHOULD-FIX-M10] LoopController "纯程序化"决策能力存疑**

- **位置:** §3.1.2 (约第 195 行) + §3.8.4 seq 7
- **问题:** LoopController 被定义为"纯程序化，不调用 LLM"，但 seq 7 的决策逻辑（"失败可重试则重新规划，失败不可重试则跳过"）需要判断失败原因的性质，这通常需要语义理解。纯规则引擎可能不够灵活。
- **建议:** 定义明确的决策规则表（失败原因 × 重试次数 → 行动），或放宽约束允许有限 LLM 调用。

**[SHOULD-FIX-M11] flaky 检测 3 次运行对大测试套件的性能影响**

- **位置:** §3.6.6 tester checklist (约第 660 行)
- **问题:** "同一测试连续 3 次运行结果一致" — 如果 pyOV 有 743 个已有测试，每次回归检查跑 3 遍就是 2229 次测试执行。加上新测试，单轮 tester 执行时间可能超过 f6 标注的 120 秒超时。
- **建议:** flaky 检测仅对新增测试执行 3 次，而非全量套件。全量套件跑 1 次即可。

---

## 四、风险点

### 🟠 High

**[MUST-FIX-H10] CR 冲突的"人工介入"与无人值守设计矛盾**

- **位置:** §3.7.5 变更冲突解决规则 (约第 1490 行)
- **问题:** 同一函数同一位置的 CR 冲突触发"人工介入（loop 暂停，通知人类）"。但 team-loop 的核心卖点是"无人值守下持续执行"（§1.2）。如果 loop 频繁暂停等待人工，就退化为人驱动模式。冲突概率在高 KR 密度时会显著上升。
- **修复建议:** 增加自动冲突解决策略（如按 DAG 优先级取后者、或跳过冲突 KR 并记录），将人工介入作为最后兜底，并设置超时自动降级。

### 🟡 Medium

**[SHOULD-FIX-M12] 2/3 投票的相关性失效风险**

- **位置:** §3.6 + §4.3
- **问题:** 2/3 投票假设 3 个 Critic 独立。但 reviewer 和 tester 可能使用同一底层 LLM（文档只说"coder≠reviewer≠tester 使用不同模型"但未强制配置）。即使不同模型，面对同类问题也可能有相关性失败。auditor 本应是独立第三方，但由于 [BLOCK-C5] 的并行问题，其实际独立性存疑。
- **建议:** 在配置中强制 3 个 Critic 使用不同 model provider；增加 auditor 对"投票一致性"的检查（3 票全相同也可能是问题）。

**[SHOULD-FIX-M13] KR 三层递进的层间门控未定义**

- **位置:** §3.5.5 (约第 430 行)
- **问题:** "必须按层递进，不可跳跃"——但 DAG 中 KR 的依赖是 task 级别的，不是层级别的。K2 的某个 KR 可以只依赖特定的 K1 KR，还是必须等所有 K1 KR 都通过？如果 K1 有 3 个 KR，K2 只需要 K1.1 通过就可以开始吗？层间门控规则未定义。
- **建议:** 明确层间门控规则：如"K2 层 KR 可在依赖的 K1 KR 通过后开始，无需全部 K1 完成"或"K2 层必须等所有 K1 层 KR 完成"。

**[SHOULD-FIX-M14] stagnation 终止条件可能过于激进**

- **位置:** §3.4 stagnation 规则 (约第 300 行)
- **问题:** "连续 3 轮覆盖率无提升" 即终止。但覆盖率提升常常是非线性的——前几轮搭建基础设施、理解代码结构，覆盖率不涨但工作有价值。3 轮即终止可能过早。
- **建议:** 考虑增加"覆盖率变化率"判断（如 delta < 阈值而非 == 0），或将阈值调到 5 轮；同时检查是否有其他进展指标（如测试可运行数增加）。

**[SHOULD-FIX-M15] 进程崩溃后 STATE.md 不一致风险**

- **位置:** §2.5.1 while 循环 + §3.7.1.2 异常捕捉
- **问题:** while 循环依赖 STATE.md 判断是否终止。但如果进程被 kill -9 或 OOM，STATE.md 不会被更新为终止状态。下次启动时，系统可能不知道上次未正常终止。§3.7.1.2 的看门狗检测到进程消失，但未说明看门狗是否更新 STATE.md。
- **建议:** 看门狗检测到进程消失时，自动在 STATE.md 写入 `FATAL: process_killed <timestamp>`；启动时先检查 STATE.md 是否有残留的 `running` 状态。

---

## 五、Low 级问题

**[NICE-L1]** §3.7.3 标题 "稳定的上下游戏离" 应为 "稳定的上下文隔离"（约第 1050 行）。明显的错别字。

**[NICE-L2]** §2.5.1 伪代码使用 `for task in 就绪的KR` 但 "就绪" 的判定标准（dependencies 全部 completed？）未在伪代码附近定义，需要读者跳转到 §3.8.4 seq 3 才能理解。

**[NICE-L3]** §2.5 与 §2.6.2 都定义了 K1/K2/K3 的含义，内容重复但措辞不同。建议统一引用 §3.5.5 作为唯一定义源。

**[NICE-L4]** §3.7.4 TaskResult 的 `output` 字段描述为"代码 diff、文件路径、执行日志"——三种不同类型的数据放在一个 string 字段中，解析困难。建议拆分为 `diff`, `files_changed`（已有）, `execution_log` 三个字段。

**[NICE-L5]** §3.8.4 seq 6 中 auditor 的 `in` 只有 `["TaskResult"]`，但 §3.1.2 定义 auditor 输入为 `TaskResult + 审查记录`。即使 [BLOCK-C5] 修复后，也需同步更新 seq 6 中 auditor 的 `in` 以包含 reviewer.Verdict 和 tester.Verdict。

---

## 总体评价

这份立项书在方法论层面非常扎实——三个核心洞察、maker/checker 分离、KR 三层模型、Ralph 技法对位等设计理念清晰且有说服力。§3.7 工程基础设施部分（日志/DI/隔离/交接物/变更管理）的考虑深度在立项书中罕见，展现了工程成熟度。

**但存在 5 个 Critical 级问题必须在编码前修复：**

1. seq 9 记忆沉淀孤岛（流程断裂）
2. §2.5 伪代码与 §3.8.4 流程结构不对应（两套真相）
3. check_order 遗漏 2 条终止规则（安全漏洞）
4. 终止条件与 STATE.md 写入机制脱节（循环可能不退出）
5. auditor 并行执行与输入契约根本矛盾（核心设计不可行）

其中第 5 个（auditor 并行问题）影响最大——它不是笔误或遗漏，而是设计层面的逻辑矛盾。auditor 被定义为"审计审查者有没有审查到位"，但流程把 auditor 与被审计者放在同一 seq 并行执行。这需要重新设计 seq 6 的结构。

**建议修复优先级：** C5 → C1 → C2 → C4 → C3 → H 级问题 → M 级问题

---

## 审查清单

- [x] 逻辑一致性: 4 个 Critical + 3 个 High — §2.5 与 §3.8.4 存在结构性矛盾
- [x] 完整性: 1 个 Critical + 4 个 High + 6 个 Medium — auditor 输入、Monitor、Schema 等多处缺失
- [x] 可执行性: 1 个 High + 5 个 Medium — KS 状态不明、工期激进、多处规格缺失
- [x] 风险点: 1 个 High + 4 个 Medium — CR 冲突、投票相关性、层间门控等
- [x] 已逐节阅读全文，行号级别意见见上文
