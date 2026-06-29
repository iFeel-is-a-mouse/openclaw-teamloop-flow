# coder 视角技术可执行性审查 — team-loop.md v1.41

> 审查时间: 2026-06-28 16:48 CST
> 审查人: coder (deepseek-v4-pro)
> 审查范围: 逐节全文，关注编码可执行性、接口完整性、技术债预警
> 审前已读取: reviewer 审查报告（5 Critical + 9 High + 12 Medium）— 不重复其发现

---

## 概述

本文档 v1.41 作为立项书极其出色——方法论扎实、工程基础设施考虑深度罕见。但从 **"明天早上我就要开始写第一行代码"** 的角度，仍有大量规格缺口需要在编码前填补。以下逐节给出具体行号级意见。

---

## 一、代码层面：可直接编码 vs 缺失规格

### 1.1 可直接编码的部分

| 位置 | 内容 | 就绪度 | 说明 |
|------|------|--------|------|
| §3.7.4 (行1151-1260) | 7 种 Pydantic Schema | 🟢 90% | `StandardGoal`、`CriticVerdict`、`MemoryEntry`、`RoundSummary` 字段完整，可直接生成 `core/types.py` |
| §3.7.5 (行1410-1510) | CR 数据模型 + 自动触发规则 | 🟢 85% | CR 模板 JSON 可直接转为 Pydantic model；4 条自动触发规则可直接翻译为 event handler |
| §3.7.1 (行750-920) | 三重输出日志框架 | 🟢 80% | 日志级别语义、文件组织、模式切换规则清晰，可直接开始 `JsonLineLogger` 和 `HumanReadableLogger` |
| §3.8.4 (行1810-1960) | roles-steps-flows 流程定义 | 🟢 75% | 12 条 flow + 9 个 seq + 6 个角色可生成 LoopController 的状态机骨架 |
| §3.7.2 (行960-1060) | DI 容器配置契约 | 🟢 75% | JSON 绑定配置 + `container.resolve()` 语义清晰 |
| §3.7.4.1 (行1260-1400) | 各 agent 模板规范 | 🟢 70% | coder/reviewer/tester/auditor/publicist/planner 的输入输出和检查清单可直接转为 prompt 模板 |
| §3.4 (行270-330) | 终止条件规则 | 🟢 70% | 7 条规则的 condition 表达式清晰（仅 check_order 漏 2 条，reviewer 已报告） |

**估算：** 以上 7 个部分共约 600-800 行可直接实现的骨架代码。

### 1.2 缺失规格 — 无法开始编码的阻塞项

#### [BLOCK-CODE-1] `KnowledgeContext` 类型从未定义（行230-250）

**位置:** §3.2.1 KS 接口概要

```python
context = ks.query(module="handlers.py", keywords=["ctx", "fixture"])  # 返回什么？
```

`KnowledgeContext` 在 §3.1.2 的 main (PlannerAgent) 输入、coder 的 TaskContext 中被引用，但**整个文档从未定义它的结构**。它包含什么字段？是文件内容片段？是 AST 节点？是 RDF 三元组列表？

**影响:** `KnowledgeProvider.query()` 接口无法定义返回类型，导致所有依赖 KS 的 agent 无法开始编码。

**建议补全:**

```python
from pydantic import BaseModel

class KnowledgeContext(BaseModel):
    """KS 查询返回的项目上下文"""
    module: str                                              # 被查询的模块名
    keywords: list[str]                                      # 匹配的关键词
    # 核心知识片段
    imports: list[ImportInfo]                                # 模块的 import 列表
    # ImportInfo: {name, alias, source_module}
    functions: list[FunctionInfo]                            # 相关函数签名
    # FunctionInfo: {name, signature, docstring, file, lineno}
    classes: list[ClassInfo]                                 # 相关类定义
    # ClassInfo: {name, methods, bases, file, lineno}
    fixtures: list[FixtureInfo]                              # 测试 fixture 定义
    # FixtureInfo: {name, scope, return_type, file, lineno}
    examples: list[ExampleInfo]                              # 测试范例
    # ExampleInfo: {test_name, test_file, approach, key_patterns}
    # 原始素材（原则1：喂原材）
    raw_snippets: list[RawSnippet]                           # 相关源码片段（原始文本，不翻译）
    # RawSnippet: {file, start_line, end_line, content, relevance_score}
    query_meta: QueryMeta                                    # 查询元信息
    # QueryMeta: {total_triples_matched, query_time_ms}
```

#### [BLOCK-CODE-2] `TaskContext` 没有正式 Pydantic 定义（行1155）

**位置:** §3.7.4 TaskUnit 定义内嵌

```python
class TaskUnit(BaseModel):
    ...
    context: TaskContext
    # TaskContext: {goal, knowledge, past_failures, constraints}  # ← 仅注释，无类型
```

**影响:** coder 收到 TaskUnit 后无法确定 `context` 字段的具体结构——它是 dict 还是 Pydantic model？`past_failures` 是什么类型？

**建议补全:**

```python
class TaskContext(BaseModel):
    """coder 执行 KR 所需的完整上下文"""
    goal: StandardGoal                                       # 当前 O
    knowledge: KnowledgeContext                              # KS 查询结果
    past_failures: list[PastFailure]                         # 历史失败记录
    constraints: list[str]                                   # 执行约束
    # ---
    # PastFailure: {task_id, description, fail_reason, critic_vetoes, round}

class PastFailure(BaseModel):
    task_id: str
    description: str
    fail_reason: str
    critic_vetoes: list[str]                                  # 哪些 critic 投了反对
    round: int
```

#### [BLOCK-CODE-3] `Task` 内嵌类型没有正式 Pydantic 定义（行1140-1150）

**位置:** §3.7.4 TaskDAG + TaskUnit 中引用

```python
class TaskDAG(BaseModel):
    ...
    tasks: list[Task]
    # Task: {task_id, description, priority, dependencies, status}  # ← 注释

class TaskUnit(BaseModel):
    ...
    task: Task
```

`Task` 和它的 `verification` 子字段在 JSON 注释中有描述（行1170-1200），但**没有 Python 类型定义**。`priority` 是 int 还是 str？`status` 有哪些合法值？`dependencies` 是 list[str] 还是 list[Task]？

**建议补全:**

```python
class KrStatus(str, Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"

class Verification(BaseModel):
    """Planner→Executor→Critic 三边契约"""
    method: Literal["pytest", "coverage", "lint", "static_analysis"]
    criteria: list[str]           # 可逐条判断 True/False 的完成条件
    evidence: str                 # 可机器读取的证据来源（文件路径/报告字段）

class Task(BaseModel):
    task_id: str
    description: str
    layer: Literal["K1", "K2", "K3"]                    # KR 三层模型
    priority: int = Field(ge=1, le=10)
    dependencies: list[str] = []                          # 依赖的 task_id
    status: KrStatus = KrStatus.PENDING
    verification: Verification
    retry_count: int = 0
    max_retries: int = 2
    created_at: str                                       # ISO 8601
    updated_at: str
```

#### [BLOCK-CODE-4] `Message` 和 `LLMResponse` 类型未定义（行970-975）

**位置:** §3.7.2 核心接口 `LLMProvider`

```python
class LLMProvider(ABC):
    @abstractmethod
    def chat(self, messages: list[Message], system: str) -> LLMResponse: ...
```

`Message` 和 `LLMResponse` 在整个文档中找不到定义。这阻止了 `LLMProvider` 接口和所有 mock 实现。

**建议补全:**

```python
class Message(BaseModel):
    role: Literal["user", "assistant", "system"]
    content: str

class LLMResponse(BaseModel):
    text: str
    usage: TokenUsage
    model: str
    finish_reason: str

class TokenUsage(BaseModel):
    input_tokens: int
    output_tokens: int
```

#### [BLOCK-CODE-5] `AgentResult` 类型未定义（行980）

**位置:** §3.7.2 AgentAdapter 接口

```python
class AgentAdapter(ABC):
    @abstractmethod
    def dispatch(self, agent_id: str, task: TaskUnit) -> AgentResult: ...
```

`AgentResult` 被多处引用但无定义。它和 `TaskResult` 的关系是什么？

**建议补全:**

```python
class AgentResult(BaseModel):
    """sessions_spawn 返回的结构"""
    run_id: str
    child_session_key: str
    agent_id: str
    status: Literal["spawned", "running", "completed", "failed", "timeout"]
```

#### [BLOCK-CODE-6] `CoverageReport` 没有正式 Schema（行1850 + 行1310）

**位置:** §3.8.4 tester cell 的 `out: ["Verdict", "CoverageReport"]`

`CoverageReport` 只在 §3.7.4.1 tester 模板中以注释形式提到字段，但 §3.7.4 的 7 种交接物（H1-H7）中不包含它。

**建议补全:**

```python
class CoverageReport(BaseModel):
    """tester 产出的覆盖率报告"""
    schema_version: str = "1.0"
    producer_agent: str = "tester"
    interaction_id: str
    round: int
    task_id: str
    timestamp: str                                           # ISO 8601
    file: str                                                # 被测文件路径
    baseline_branch_coverage: float                          # 执行前覆盖率 %
    new_branch_coverage: float                               # 执行后覆盖率 %
    delta: float                                             # 覆盖率增量（可负）

# 此类型应作为 H8 加入 §3.7.4
```

---

## 二、接口完整性分析

### 2.1 6 个 ABC：当前声明 vs 实际需要的差距

#### `LLMProvider` — 缺 2 个方法

```python
# 当前声明（行970-975）
class LLMProvider(ABC):
    @abstractmethod
    def chat(self, messages: list[Message], system: str) -> LLMResponse: ...
    @abstractmethod
    def model_name(self) -> str: ...

# 实际需要补充的方法：
    @abstractmethod
    def chat_stream(self, messages: list[Message], system: str) -> Iterator[str]:
        """流式输出，用于需要实时响应的场景"""
        ...

    @abstractmethod
    def token_count(self, messages: list[Message]) -> int:
        """估算消息的 token 数。预算熔断（§3.4 budget_exhausted）依赖此方法"""
        ...
```

**token_count 为什么必要：** §3.4 budget_exhausted 条件需要 `estimated_next_round_tokens`。如果不提供 token 计数接口，预算熔断只能依赖 LLM 返回的 usage 字段（事后统计），而非事前估算。

#### `KnowledgeProvider` — 缺 2 个方法 + 返回值定义

```python
# 当前声明（行978-981）
class KnowledgeProvider(ABC):
    @abstractmethod
    def query(self, module: str, keywords: list[str]) -> KnowledgeContext: ...
    @abstractmethod
    def index(self, project_path: str) -> None: ...

# 实际需要补充的方法：
    @abstractmethod
    def query_raw(self, sparql: str) -> list[dict]:
        """执行原始 SPARQL 查询，返回绑定列表。
        PlannerAgent 需要此能力做复杂检索（如"查找所有使用 ctx 的测试"）"""
        ...

    @abstractmethod
    def health_check(self) -> bool:
        """KS 可达性检查。§3.4 ks_unreachable 终止规则依赖此方法"""
        ...
```

#### `AgentAdapter` — 返回值类型有歧义

```python
# 当前声明（行983-986）
class AgentAdapter(ABC):
    @abstractmethod
    def dispatch(self, agent_id: str, task: TaskUnit) -> AgentResult: ...
    @abstractmethod
    def await_result(self, task_id: str, timeout_sec: int) -> TaskResult: ...

# 问题：dispatch 是非阻塞调用（async），立即返回 runId。
# 但 `AgentResult` vs `TaskResult` 的关系未定义。
# 建议明确区分：

    @abstractmethod
    def dispatch(self, agent_id: str, task: TaskUnit) -> DispatchResult:
        """非阻塞下发，立即返回 run_id（对应 OpenClaw sessions_spawn 返回的 childSessionKey）"""
        ...

    @abstractmethod
    def await_result(self, dispatch_id: str, timeout_sec: int) -> TaskResult | None:
        """阻塞等待子 agent 完成，返回 TaskResult 或超时返回 None"""
        ...

    @abstractmethod
    def cancel(self, dispatch_id: str) -> bool:
        """取消一个进行中的子 agent。agent_timeout 终止规则需要此能力"""
        ...
```

#### `StoreProvider` — 缺 `append` 和 `delete` 方法

```python
# 当前声明（行988-993）
class StoreProvider(ABC):
    @abstractmethod
    def get(self, collection: str, key: str) -> dict | None: ...
    @abstractmethod
    def set(self, collection: str, key: str, value: dict) -> None: ...
    @abstractmethod
    def list(self, collection: str, filter_fn=None) -> list[dict]: ...

# 实际需要补充的方法（参考 Loop Engineering 参考实现）：
    @abstractmethod
    def delete(self, collection: str, key: str) -> None:
        """删除条目。CR 回滚（§3.7.5）需要此能力"""
        ...

    @abstractmethod
    def append(self, collection: str, key: str, value: dict) -> None:
        """追加到列表字段。日志追加、CR 追加需要此能力"""
        ...

    @abstractmethod
    def keys(self, collection: str) -> list[str]:
        """列出 collection 中所有 key。LoopController 需要枚举所有 task"""
        ...
```

#### `LoggerProvider` — `log` 方法参数过多，建议拆分

```python
# 当前声明（行995-1000）
class LoggerProvider(ABC):
    @abstractmethod
    def log(self, level: str, agent: str, event: str,
            interaction_id: str, loop_id: str, round: int,
            task_id: str | None, data: dict) -> None: ...

# 问题：8 个位置参数违反 NASA 规则（>4）。每次调用需要重复传 interaction_id/loop_id/round。
# 建议引入 LogContext 对象：

class LogContext(BaseModel):
    interaction_id: str
    loop_id: str
    round: int
    task_id: str | None = None

class LoggerProvider(ABC):
    @abstractmethod
    def log(self, level: str, agent: str, event: str, ctx: LogContext, data: dict) -> None: ...

    # 便捷方法 — 减少调用方心智负担
    def info(self, agent: str, event: str, ctx: LogContext, data: dict = {}) -> None:
        self.log("INFO", agent, event, ctx, data)

    def error(self, agent: str, event: str, ctx: LogContext, data: dict = {}) -> None:
        self.log("ERROR", agent, event, ctx, data)
```

#### `EventBus` — 基本完整，缺 `unsubscribe`

```python
class EventBus(ABC):
    @abstractmethod
    def publish(self, event: str, data: dict) -> None: ...
    @abstractmethod
    def subscribe(self, event: str, handler: Callable) -> None: ...

    # 建议补充：
    @abstractmethod
    def unsubscribe(self, event: str, handler: Callable) -> None:
        """取消订阅。长时间运行的 loop 需要避免 handler 累积"""
        ...

    @abstractmethod
    def once(self, event: str, handler: Callable) -> None:
        """一次性订阅，触发后自动取消"""
        ...
```

### 2.2 pyrdf-knowledge API 缺口

KS 只有 3 个方法（`index`、`query`、`insert`），文档未展开。从 team-loop 的使用场景倒推，KS 至少需要：

```python
class KnowledgeServer:
    """项目知识伺服 — 完整 API 契约（建议）"""

    # ── 索引 ──
    def index(self, project_path: str) -> IndexResult:
        """扫描项目源码，生成 rdf/ast/*.ttl
        返回：IndexResult {files_scanned, triples_added, duration_ms, errors[]}
        """
        ...

    def reindex(self, changed_files: list[str]) -> IndexResult:
        """增量重索引。coder 修改文件后增量更新，不全量重扫"""
        ...

    # ── 查询 ──
    def query(self, module: str, keywords: list[str],
              max_results: int = 20) -> KnowledgeContext:
        """语义查询 — 基于 SPARQL，对调用方屏蔽 RDF 细节"""
        ...

    def sparql(self, query_str: str) -> list[dict[str, str]]:
        """原始 SPARQL 查询 — 供高级场景（如查找所有使用 pytest.fixture 的测试）"""
        ...

    def search_examples(self, module: str, function_pattern: str | None = None
                        ) -> list[ExampleInfo]:
        """搜索测试范例 — PlannerAgent 拆解 KR 时核心依赖"""
        ...

    # ── 记忆 ──
    def insert_memory(self, entry: MemoryEntry) -> None:
        """将 MemoryEntry 写入 rdf/ast/memory.ttl"""
        ...

    def query_memory(self, keywords: list[str], limit: int = 10) -> list[MemoryEntry]:
        """搜索历史记忆，按相关性排序"""
        ...

    # ── 运维 ──
    def health_check(self) -> HealthStatus:
        """返回：{reachable, triple_count, last_index_time, errors[]}"""
        ...

    def stats(self) -> KsStats:
        """返回：{total_triples, files_indexed, index_duration_ms, memory_entries}"""
        ...
```

**关键的 KS 实现细节缺失：**

1. **RDF Schema/Ontology 未定义。** 文档说生成 `rdf/ast/*.ttl`，但 Turtle 文件中使用什么谓词？是自定 ontology 还是复用标准词汇（如 `doap:`、`schema:`）？编码前必须至少定义核心谓词表。
2. **`query` 如何将 keyword 翻译为 SPARQL。** 文档给的关键词例子是 `["ctx", "fixture"]`——如何从这两个词生成有效的 SPARQL 查询？需要说明查询翻译策略。
3. **索引结果的缓存与失效。** `index()` 扫描源码生成 ttl，如果源码变更但未 reindex，`query()` 返回的是过期数据。需要设计失效策略。

### 2.3 adapters/ 实现清单

| 适配器 | 实现接口 | 关键实现细节 |
|--------|---------|------------|
| `openclaw.py` | `AgentAdapter` | 封装 `sessions_spawn` + push event 等待 |
| `direct_call.py` | `AgentAdapter` | 同步函数调用，测试用 |
| `deepseek.py` | `LLMProvider` | 封装 DeepSeek API |
| `glm.py` | `LLMProvider` | 封装 GLM API |
| `rdf.py` | `KnowledgeProvider` | 封装 pyrdf-knowledge |
| `json_store.py` | `StoreProvider` | JSON 文件持久化（参考 Loop Eng. 实现） |
| `memory_store.py` | `StoreProvider` | 内存 dict，测试用 |
| `jsonline_logger.py` | `LoggerProvider` | §3.7.1 JSON 行输出 |
| `human_logger.py` | `LoggerProvider` | §3.7.1 彩色终端输出 |
| `metrics_logger.py` | `LoggerProvider` | §3.7.1 运维指标输出 |

**缺失的适配器：** `memory_eventbus.py`（EventBus 的内存实现）

---

## 三、技术债预警

### 3.1 🔴 高风险：KS 索引与「不翻译」原则的矛盾

**位置:** §2.2 原则 1 vs §3.2.1 KS index()

文档 §2.2 核心洞察 1 明确说：**"不翻译，喂原材 — LLM 的代码理解能力远超任何手写解析器，不要 AST 中间翻译层"**。但 §3.2.1 的 `ks.index()` 设计是"扫描源码生成 `rdf/ast/*.ttl`"——这正是 AST 翻译层，将源码转成 RDF 三元组。

**这个矛盾的后果取决于实现路径：**

- **路径 A（AST→RDF 翻译）：** 需要手写 Python/C 语法解析器，将 AST 节点转成 RDF 三元组。这是 debugloop 被评价为"增加复杂度"的方案。复杂度高、维护成本大、对不同语言的代码需要不同解析器。
- **路径 B（原始文件索引+关键词匹配）：** 不做 AST 翻译，而是建立文件名→函数名→行号的倒排索引，`query` 返回原始源码片段。这符合"喂原材"原则，但 query("ctx") 返回什么？返回所有包含 "ctx" 的代码行？精度低。

**建议在启动编码前明确选择路径：**

```
路径 B（推荐，与原则1一致）：
  KS 不做 AST→RDF 翻译。
  index() 建立：
    - 文件名 → 文件内容 映射（原始文本）
    - 符号名 → 定义位置 映射（如 "handle_tree" → "handlers.py:120-145"）
    - 关键词 → 出现位置 倒排索引
  query(keywords=["ctx", "fixture"]) →
    1. 倒排索引查 "ctx" → 找到 handlers.py:120-145
    2. 返回原始代码片段（直接截取行120-145的文本）
    3. 不翻译、不总结、不重构

  如果后续需要语义检索（如"查找所有 HTTP 测试"），
  用 embedding 向量 + 关键词混合搜，而非手写 AST。
```

### 3.2 🟡 中风险：rdflib 的 SPARQL 实现性能

即使 KS 选择路径 B（不用 rdflib），如果记忆存储使用 `rdf/ast/memory.ttl` 格式，仍涉及 rdflib。

**已知问题：**
- rdflib 的 SPARQL 引擎是纯 Python 实现，对于 >1000 条三元组的图，`FILTER` 和 `OPTIONAL` 查询可能耗时秒级
- rdflib 的 `Graph.query()` 不是线程安全的——如果 publicist 写 memory.ttl 的同时 planner 做 SPARQL 查询，需要外部锁
- Turtle 文件的解析/序列化对大图（>5000 三元组）O(n²) 内存增长

**缓解建议：**
```python
# 1. 使用 rdflib-sqlalchemy 或 oxigraph 作为 rdflib 的存储后端
# 2. 或者完全放弃 RDF，改用 SQLite + 全文搜索（FTS5），更简单且性能更好
# 3. 如果坚持 rdflib，加读写锁

import threading

class RdfMemoryStore:
    def __init__(self):
        self._graph = rdflib.Graph()
        self._lock = threading.RWLock()  # 读写锁

    def insert(self, entry: MemoryEntry):
        with self._lock.writer():
            # 写操作
            ...

    def query(self, sparql: str) -> list:
        with self._lock.reader():
            # 读操作
            ...
```

### 3.3 🟡 中风险：STATE.md 文件轮询的可靠性

**位置:** §2.5.1 while 循环条件 + 附录 D `.team-loop/STATE.md`

`grep -q "终止" STATE.md` 作为循环终止机制存在以下问题：

1. **竞争条件：** 写 STATE.md 和读 STATE.md 不在同一进程——可能出现写入未 flush、读取到部分写入的内容
2. **进程崩溃残留：** kill -9 后 STATE.md 保持 "running"，下次启动误认为 loop 仍在运行
3. **无锁机制：** 多个组件可能同时写入 STATE.md

**建议实现：**

```python
# 使用原子写入 + 状态机
class LoopStateManager:
    STATE_FILE = ".team-loop/STATE.json"

    VALID_STATES = {"running", "SUCCESS", "FATAL", "MAX_ROUNDS", "BUDGET", "STAGNATION"}

    @classmethod
    def read_state(cls) -> dict:
        """读取状态，含启动时残留检测"""
        if not os.path.exists(cls.STATE_FILE):
            return {"state": "running", "loop_id": None}
        with open(cls.STATE_FILE, "r") as f:
            data = json.load(f)
        # 残留检测：如果状态是 running 但进程已不存在
        if data.get("state") == "running":
            pid = data.get("pid")
            if pid and not cls._is_process_alive(pid):
                data["state"] = "FATAL"
                data["reason"] = f"Previous process PID {pid} no longer alive"
        return data

    @classmethod
    def write_state(cls, state: str, **kwargs):
        """原子写入：先写临时文件再 rename"""
        if state not in cls.VALID_STATES:
            raise ValueError(f"Invalid state: {state}")
        data = {"state": state, "pid": os.getpid(),
                "timestamp": datetime.now().isoformat(), **kwargs}
        tmp = cls.STATE_FILE + ".tmp"
        with open(tmp, "w") as f:
            json.dump(data, f)
            f.flush()
            os.fsync(f.fileno())
        os.rename(tmp, cls.STATE_FILE)  # 原子操作
```

### 3.4 🟡 中风险：LoopController「纯程序化」决策的能力边界

**位置:** §3.1.2 LoopController 约束 "纯程序化，不调用 LLM" + §3.8.4 seq 7

seq 7 需要判断"失败可重试则重新规划，失败不可重试则跳过"。这个判断需要语义理解——coder 写错了一个 import vs coder 生成了幻觉 API，前者可重试后者应跳过。

**纯规则引擎的决策表必须预先定义所有分支：**

```python
# 推荐的决策规则表
RETRY_RULES = {
    # (fail_reason_category, retry_count_condition) → action
    ("import_error", "retry_count < max_retries"): "retry",
    ("api_hallucination", "*"): "skip",       # 幻觉不可重试
    ("coverage_not_met", "retry_count < 1"): "retry",
    ("test_setup_crash", "retry_count < max_retries"): "retry",
    ("regression_detected", "*"): "skip",      # 回归不可重试
    ("critic_disagree", "retry_count == 0"): "retry",  # 3票中有分歧，给一次重试
    ("timeout", "retry_count == 0"): "retry",
    ("*", "retry_count >= max_retries"): "skip",  # 兜底
}
```

**建议：** 要么提供完整的决策表（如上），要么放宽 `LoopController` 的约束，允许在 seq 7 决策时调用一次 LLM（token 成本可控，远小于错误重试的消耗）。

### 3.5 🟡 中风险：timeout 的实际行为未定义

**位置:** §3.8.4 f3 [超时300秒]、f5-f7 [超时120秒]

当 sessions_spawn 超过 timeout 时，实际发生什么？

- OpenClaw 是否自动 kill 子 session？还是只返回 timeout 事件？
- main 收到 timeout 后：算失败？算跳过？触发 f12 反馈环还是 seq 8 终止判断？
- 超时的 coder 可能已经写入部分文件——tester 看到的是部分产物还是空？

**建议在 `AgentAdapter` 接口中明确：**

```python
class TimeoutPolicy(Enum):
    CANCEL_AND_FAIL = "cancel_and_fail"       # kill 子进程，标记失败
    CANCEL_AND_RETRY = "cancel_and_retry"     # kill 子进程，重试
    GRACEFUL_RETURN = "graceful_return"       # 不 kill，等待子进程自然结束但标记超时

class AgentAdapter(ABC):
    ...
    @abstractmethod
    def dispatch(self, agent_id: str, task: TaskUnit,
                 timeout_sec: int,
                 on_timeout: TimeoutPolicy = TimeoutPolicy.CANCEL_AND_FAIL
                 ) -> DispatchResult: ...
```

### 3.6 🟢 低风险：CR 编号冲突（并发场景）

**位置:** §3.7.5 CR 编号规则 `CR-{interaction_id_short}-{seq}`

如果 loop 中有多个 KR 并行执行（如 K1 的 3 个 KR 同时由 coder 完成），它们可能同时写入 CR 记录。自增 seq 在文件系统中不是原子的。

**建议:** 用 UUID 短码或毫秒时间戳替代自增序号：

```python
cr_id = f"CR-{interaction_id_short}-{uuid4().hex[:6]}"  # 如 CR-a3f2-b7e1d4
```

---

## 四、具体行动建议（带伪代码）

### 4.1 立即可写的代码骨架

以下是 `core/interfaces.py` 的推荐实现（解决了上述所有接口缺口）：

```python
"""team-loop 核心接口定义 — 纯 ABC，无实现依赖"""
from abc import ABC, abstractmethod
from typing import Callable, Iterator, Literal
from pydantic import BaseModel, Field
from enum import Enum
from datetime import datetime


# ═══════════════════════════════════════════════════
# 基础类型（§3.7.4 缺的类型在此补全）
# ═══════════════════════════════════════════════════

class TokenUsage(BaseModel):
    input_tokens: int
    output_tokens: int

class Message(BaseModel):
    role: Literal["user", "assistant", "system"]
    content: str

class LLMResponse(BaseModel):
    text: str
    usage: TokenUsage
    model: str = ""
    finish_reason: str = "stop"

class KrStatus(str, Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"

class Verification(BaseModel):
    method: Literal["pytest", "coverage", "lint", "static_analysis"]
    criteria: list[str]
    evidence: str

class Task(BaseModel):
    task_id: str
    description: str
    layer: Literal["K1", "K2", "K3"]
    priority: int = Field(ge=1, le=10)
    dependencies: list[str] = []
    status: KrStatus = KrStatus.PENDING
    verification: Verification
    retry_count: int = 0
    max_retries: int = 2
    created_at: str
    updated_at: str

class ImportInfo(BaseModel):
    name: str
    alias: str | None = None
    source_module: str | None = None

class FunctionInfo(BaseModel):
    name: str
    signature: str
    docstring: str | None = None
    file: str
    lineno: int

class FixtureInfo(BaseModel):
    name: str
    scope: str
    return_type: str | None = None
    file: str
    lineno: int

class RawSnippet(BaseModel):
    file: str
    start_line: int
    end_line: int
    content: str
    relevance_score: float

class QueryMeta(BaseModel):
    total_triples_matched: int
    query_time_ms: int

class KnowledgeContext(BaseModel):
    module: str
    keywords: list[str]
    imports: list[ImportInfo] = []
    functions: list[FunctionInfo] = []
    fixtures: list[FixtureInfo] = []
    raw_snippets: list[RawSnippet] = []
    query_meta: QueryMeta | None = None

class PastFailure(BaseModel):
    task_id: str
    description: str
    fail_reason: str
    critic_vetoes: list[str]
    round: int

class TaskContext(BaseModel):
    goal: "StandardGoal"
    knowledge: KnowledgeContext
    past_failures: list[PastFailure] = []
    constraints: list[str] = []

class LogContext(BaseModel):
    interaction_id: str
    loop_id: str
    round: int
    task_id: str | None = None

# ═══════════════════════════════════════════════════
# 6 个核心接口（补全方法签名）
# ═══════════════════════════════════════════════════

class LLMProvider(ABC):
    @abstractmethod
    def chat(self, messages: list[Message], system: str = "") -> LLMResponse: ...
    @abstractmethod
    def model_name(self) -> str: ...
    @abstractmethod
    def token_count(self, messages: list[Message]) -> int:
        """估算 token 数，供预算熔断使用"""
        ...

class KnowledgeProvider(ABC):
    @abstractmethod
    def query(self, module: str, keywords: list[str],
              max_results: int = 20) -> KnowledgeContext: ...
    @abstractmethod
    def query_raw(self, sparql: str) -> list[dict[str, str]]: ...
    @abstractmethod
    def index(self, project_path: str) -> dict:
        """返回 {files_scanned, triples_added, duration_ms, errors}"""
        ...
    @abstractmethod
    def reindex(self, changed_files: list[str]) -> dict: ...
    @abstractmethod
    def insert_memory(self, entry: "MemoryEntry") -> None: ...
    @abstractmethod
    def query_memory(self, keywords: list[str], limit: int = 10) -> list["MemoryEntry"]: ...
    @abstractmethod
    def health_check(self) -> bool: ...

class DispatchResult(BaseModel):
    run_id: str
    child_session_key: str
    agent_id: str
    status: Literal["spawned"]

class TimeoutPolicy(str, Enum):
    CANCEL_AND_FAIL = "cancel_and_fail"
    CANCEL_AND_RETRY = "cancel_and_retry"
    GRACEFUL_RETURN = "graceful_return"

class AgentAdapter(ABC):
    @abstractmethod
    def dispatch(self, agent_id: str, task: "TaskUnit",
                 timeout_sec: int = 300,
                 on_timeout: TimeoutPolicy = TimeoutPolicy.CANCEL_AND_FAIL
                 ) -> DispatchResult: ...
    @abstractmethod
    def await_result(self, dispatch_id: str,
                     timeout_sec: int) -> "TaskResult | None": ...
    @abstractmethod
    def cancel(self, dispatch_id: str) -> bool: ...

class StoreProvider(ABC):
    @abstractmethod
    def get(self, collection: str, key: str) -> dict | None: ...
    @abstractmethod
    def set(self, collection: str, key: str, value: dict) -> None: ...
    @abstractmethod
    def list(self, collection: str,
             filter_fn: Callable[[dict], bool] | None = None) -> list[dict]: ...
    @abstractmethod
    def delete(self, collection: str, key: str) -> None: ...
    @abstractmethod
    def append(self, collection: str, key: str, value: dict) -> None: ...
    @abstractmethod
    def keys(self, collection: str) -> list[str]: ...

class LoggerProvider(ABC):
    @abstractmethod
    def log(self, level: str, agent: str, event: str,
            ctx: LogContext, data: dict | None = None) -> None: ...
    def debug(self, agent: str, event: str, ctx: LogContext,
              data: dict | None = None) -> None:
        self.log("DEBUG", agent, event, ctx, data or {})
    def info(self, agent: str, event: str, ctx: LogContext,
             data: dict | None = None) -> None:
        self.log("INFO", agent, event, ctx, data or {})
    def warn(self, agent: str, event: str, ctx: LogContext,
             data: dict | None = None) -> None:
        self.log("WARN", agent, event, ctx, data or {})
    def error(self, agent: str, event: str, ctx: LogContext,
              data: dict | None = None) -> None:
        self.log("ERROR", agent, event, ctx, data or {})
    def fatal(self, agent: str, event: str, ctx: LogContext,
              data: dict | None = None) -> None:
        self.log("FATAL", agent, event, ctx, data or {})

class EventBus(ABC):
    @abstractmethod
    def publish(self, event: str, data: dict) -> None: ...
    @abstractmethod
    def subscribe(self, event: str, handler: Callable[[dict], None]) -> None: ...
    @abstractmethod
    def unsubscribe(self, event: str, handler: Callable[[dict], None]) -> None: ...
    @abstractmethod
    def once(self, event: str, handler: Callable[[dict], None]) -> None: ...
```

### 4.2 LoopController 核心骨架

```python
"""loopctrl.py — LoopController 主循环（纯程序化）"""

class TerminationReason(Enum):
    ALL_KR_COMPLETED = "all_kr_completed"
    BASIC_LAYER_FATAL = "basic_layer_fatal"
    BUDGET_EXHAUSTED = "budget_exhausted"
    MAX_ROUNDS = "max_rounds"
    STAGNATION = "stagnation"
    KS_UNREACHABLE = "ks_unreachable"
    AGENT_TIMEOUT = "agent_timeout"

class Decision(Enum):
    NEXT = "next"          # 继续下一轮
    RETRY = "retry"        # 重试当前 KR
    SKIP = "skip"          # 跳过当前 KR
    TERMINATE = "terminate" # 终止

class LoopController:
    def __init__(self, llm: LLMProvider, knowledge: KnowledgeProvider,
                 agent: AgentAdapter, store: StoreProvider,
                 logger: LoggerProvider, eventbus: EventBus):
        self.llm = llm
        self.knowledge = knowledge
        self.agent = agent
        self.store = store
        self.logger = logger
        self.eventbus = eventbus

    def judge_termination(self, state: "LoopState") -> tuple[Decision, TerminationReason | None]:
        """§3.4 终止条件判断 — check_order 优先级逐条判定"""

        checks = [
            ("all_kr_completed",      self._check_all_kr_completed),
            ("basic_layer_fatal",     self._check_basic_layer_fatal),
            ("ks_unreachable",        self._check_ks_unreachable),     # ← reviewer C3: 补入
            ("agent_timeout",         self._check_agent_timeout),      # ← reviewer C3: 补入
            ("budget_exhausted",      self._check_budget_exhausted),
            ("max_rounds",            self._check_max_rounds),
            ("stagnation",            self._check_stagnation),
        ]

        for rule_id, check_fn in checks:
            terminated, reason = check_fn(state)
            if terminated:
                return Decision.TERMINATE, reason

        return Decision.NEXT, None

    def decide_retry_or_skip(self, verdict: "AggregatedVerdict",
                             task: Task) -> Decision:
        """§3.8.4 seq 7 — 2/3 裁决后的行动决策（纯规则表）"""
        if verdict.passed_count >= 2:
            return Decision.NEXT

        if task.retry_count >= task.max_retries:
            return Decision.SKIP

        # 不可重试的失败类型
        non_retryable = {"api_hallucination", "regression_detected",
                         "dependency_failed"}
        if any(r in non_retryable for r in verdict.fail_reasons):
            return Decision.SKIP

        return Decision.RETRY

    # 以下为各 check 的私有实现...
    def _check_all_kr_completed(self, state):
        pending = sum(1 for t in state.tasks if t.status == KrStatus.PENDING)
        in_progress = sum(1 for t in state.tasks if t.status == KrStatus.IN_PROGRESS)
        if pending == 0 and in_progress == 0:
            return True, TerminationReason.ALL_KR_COMPLETED
        return False, None

    def _check_basic_layer_fatal(self, state):
        k1_tasks = [t for t in state.tasks if t.layer == "K1"]
        consecutive = 0
        for t in reversed(k1_tasks):
            if t.status == KrStatus.FAILED:
                consecutive += 1
            else:
                break
        if consecutive >= 3:
            return True, TerminationReason.BASIC_LAYER_FATAL
        return False, None

    def _check_stagnation(self, state):
        if len(state.coverage_history) < 3:
            return False, None
        last_3 = state.coverage_history[-3:]
        if all(delta == 0 for delta in last_3):
            return True, TerminationReason.STAGNATION
        return False, None

    # ... (其他 check 方法类似)
```

### 4.3 Logger 简化实现（三重输出）

```python
"""日志框架核心 — §3.7.1"""

class LogRouter:
    """路由到三个 Logger 适配器，agent 不感知输出格式"""
    def __init__(self, jsonl: JsonLineLogger, human: HumanReadableLogger,
                 metrics: MetricsLogger):
        self._outputs = [jsonl, human, metrics]
        self._mode = "looplog"  # looplog | lean

    def log(self, level: str, agent: str, event: str,
            ctx: LogContext, data: dict | None = None):
        data = data or {}
        ts = datetime.now(timezone.utc).isoformat()

        # Lean 模式下跳过 DEBUG 和部分 INFO
        if self._mode == "lean" and level in ("DEBUG",):
            return
        if self._mode == "lean" and level == "INFO":
            # Lean 模式下保留 WARN+ 的人类展开
            pass

        for output in self._outputs:
            output.write(level, agent, event, ctx, data, ts)

    def set_mode(self, mode: str):
        if mode not in ("looplog", "lean"):
            raise ValueError(f"Invalid log mode: {mode}")
        self._mode = mode


class JsonLineLogger(LoggerProvider):
    """JSON 行输出 — 给 loop 自身消费"""
    def __init__(self, log_dir: str):
        self._log_dir = Path(log_dir)

    def log(self, level, agent, event, ctx, data, ts):
        path = self._log_dir / ctx.loop_id / f"R{ctx.round:03d}" / f"{agent}.jsonl"
        path.parent.mkdir(parents=True, exist_ok=True)
        record = {"ts": ts, "level": level, "agent": agent, "event": event,
                  "interaction_id": ctx.interaction_id, "loop_id": ctx.loop_id,
                  "round": ctx.round, "task_id": ctx.task_id, "data": data}
        with open(path, "a") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")


class HumanReadableLogger(LoggerProvider):
    """彩色终端 + 文件输出 — 给开发者"""
    COLORS = {"DEBUG": "\033[90m", "INFO": "\033[0m", "WARN": "\033[33m",
              "ERROR": "\033[31m", "FATAL": "\033[31;1m"}
    RESET = "\033[0m"

    def log(self, level, agent, event, ctx, data, ts):
        header = f"[{ts[:19]}] [{level:<5}] [{agent:<8}] {ctx.loop_id} R{ctx.round}"
        if ctx.task_id:
            header += f" {ctx.task_id}"
        line = f"{self.COLORS.get(level, '')}{header} │ {event}{self.RESET}"
        print(line)
        if data and level in ("WARN", "ERROR", "FATAL"):
            for k, v in data.items():
                print(f"                                                     │    {k}: {v}")
```

### 4.4 KS（Knowledge Server）最小可行实现（路径 B）

```python
"""ks.py — 项目知识伺服，最小可行实现（路径B：原始文件索引）"""

class KnowledgeServer:
    def __init__(self, project_path: str):
        self._project = Path(project_path)
        self._index: dict[str, dict] = {}  # module → {content, symbols}

    def index(self, project_path: str | None = None) -> dict:
        """扫描源码，建立倒排索引。不翻译 AST。"""
        if project_path:
            self._project = Path(project_path)
        src_dir = self._project / "src"
        test_dir = self._project / "tests"
        files_scanned = 0
        errors = []

        for py_file in chain(src_dir.rglob("*.py"), test_dir.rglob("*.py")):
            try:
                rel_path = str(py_file.relative_to(self._project))
                content = py_file.read_text()
                self._index[rel_path] = {
                    "content": content,
                    "lines": content.splitlines(),
                    "defs": self._extract_defs(content),  # 简单的 regex 提取
                }
                files_scanned += 1
            except Exception as e:
                errors.append(str(e))

        return {"files_scanned": files_scanned, "errors": errors}

    @staticmethod
    def _extract_defs(content: str) -> list[dict]:
        """用简单 regex 提取函数/类定义，不建 AST"""
        import re
        defs = []
        for line_no, line in enumerate(content.splitlines(), 1):
            m = re.match(r'^\s*def\s+(\w+)\((.*?)\)', line)
            if m:
                defs.append({"type": "function", "name": m.group(1),
                             "signature": m.group(2), "line": line_no})
            m = re.match(r'^\s*class\s+(\w+)', line)
            if m:
                defs.append({"type": "class", "name": m.group(1), "line": line_no})
            m = re.match(r'^\s*@pytest\.fixture', line)
            if m:
                defs.append({"type": "fixture_annotation", "line": line_no})
        return defs

    def query(self, module: str, keywords: list[str],
              max_results: int = 20) -> KnowledgeContext:
        """关键词匹配 + 原始源码片段返回（原则1：喂原材）"""
        if module not in self._index:
            return KnowledgeContext(module=module, keywords=keywords)

        entry = self._index[module]
        snippets = []

        for kw in keywords:
            for i, line in enumerate(entry["lines"], 1):
                if kw in line:
                    # 取上下文（前后各 3 行）
                    start = max(0, i - 4)
                    end = min(len(entry["lines"]), i + 3)
                    context = "\n".join(
                        f"{j}: {entry['lines'][j-1]}"
                        for j in range(start + 1, end + 1)
                    )
                    snippets.append(
                        RawSnippet(file=module, start_line=start + 1,
                                   end_line=end, content=context,
                                   relevance_score=1.0)
                    )

        # 去重 + 排序 + 截断
        seen = set()
        unique = []
        for s in sorted(snippets, key=lambda x: -x.relevance_score):
            key = (s.file, s.start_line, s.end_line)
            if key not in seen:
                seen.add(key)
                unique.append(s)

        return KnowledgeContext(
            module=module, keywords=keywords,
            raw_snippets=unique[:max_results],
            query_meta=QueryMeta(total_triples_matched=len(snippets),
                                 query_time_ms=0)
        )
```

---

## 五、总结与优先级建议

### 编码前必须完成（阻塞项，约 1 天）

| 优先级 | 任务 | 产出 |
|--------|------|------|
| P0 | 补全 §3.7.4 缺失类型：`Task`、`TaskContext`、`KnowledgeContext`、`Message`、`LLMResponse`、`AgentResult`、`CoverageReport`、`PastFailure` 的 Pydantic 定义 | 更新 team-loop.md 或写入 `types-design.md` |
| P0 | 补全 6 个 ABC 的完整方法签名（含返回值类型、异常规范）| `core/interfaces.py` 初版 |
| P0 | 决定 KS 实现路径（RDF/AST 翻译 vs 原始文件索引），与原则 1 对齐 | 更新 §3.2.1 或 ADR |
| P0 | 定义 KR 状态机（pending→in_progress→completed/failed/skipped 转换规则）| 状态转换图 + 枚举 |
| P0 | 定义重试决策表（哪些失败可重试、max_retries 默认值）| 决策表 |

### 编码开始时优先实现（基础件，约 1.5 天）

| 优先级 | 任务 | 理由 |
|--------|------|------|
| P1 | `core/types.py` — 全部 Pydantic Schema | 所有模块的输入输出依赖 |
| P1 | `core/interfaces.py` — 6 个 ABC + 辅助类型 | DI 容器和 mock 的前提 |
| P1 | `core/events.py` — 事件类型枚举 | EventBus 的前提 |
| P1 | Mock 全套（6 个接口的实现） | 先跑通虚拟 loop，再对接真实环境 |
| P1 | Logger 三重输出（JsonLine + Human + Metrics） | 所有 agent 运行时依赖日志 |

### 编码中需注意的技术决策点

1. **KS 实现：优先选择路径 B（原始文件索引 + 关键词匹配），放弃 rdflib。** 与原则 1 "不翻译"一致、实现成本低、维护简单。记忆存储用 SQLite + FTS5 或简单 JSON 文件，不用 RDF。
2. **`sessions_spawn` 的 TaskUnit 传递：** 将 TaskUnit JSON 序列化到 `task` 字符串参数，或写入临时文件让子 agent `read`。建议用 task 字符串 + `attachments` 传递大上下文。
3. **覆盖率的基线和增量计算：** 在 `rdf/` 或 `.team-loop/` 下存储 `baseline_coverage.json`，每轮 tester 执行后更新。增量 = 新 coverage - 基线。
4. **`@traceable` 装饰器：** 用 `functools.wraps` + `time.perf_counter()` 实现，记录 `{function, args_hash, start_time, end_time, duration_ms, status, error}`。避免引入 OpenTelemetry 等重型依赖。

---

> **总体评价：** 此立项书工程深度罕见。解决了以上阻塞项（约 1 天工作）后，135 行 §4.1 中列出的 7 个部分可以并行开始编码。预计 Step 1-4（接口→mock→核心逻辑）从 3 天压缩到 2.5 天是可行的，前提是 P0 阻塞项在编码前完成。

> **与 reviewer 报告的关系：** 审查中发现 reviewer 报告的 5 Critical / 9 High 问题与本报告的 P0 阻塞项是独立维度——reviewer 关注逻辑一致性，本报告关注代码可执行性。两者需共同修复后方可进入编码阶段。
