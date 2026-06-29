# debugloop 项目上下文自动注入方案

> 思路与可行性分析 | 2026-06-27
>
> 问题来源：debugloop 在 pyOV 项目上基线 33.9%，2 轮迭代覆盖率零提升，handler 层 ~942 个分支纹丝不动。3 个 LLM 生成的测试全部失败。

---

## 1. 问题诊断

### 1.1 debugloop 当前给 LLM 喂了什么

看了 `prompts.py` 的 prompt 构造逻辑，debugloop 目前向 LLM 注入的信息包括：

- **未覆盖分支详情** — 文件路径、行号、分支类型、目标代码片段、所在函数的完整源码（截断到 80 行）
- **API 签名清单** — 由 `build_api_manifest()` 扫描项目 AST 生成，包含每个模块的函数/类/方法签名
- **项目导入路径** — `_discover_project_imports()` 扫描出的可导入符号
- **设计文档** — `_load_design_docs()` 读取 `docs/` 下的 .md 文件前 3000 字符
- **函数调用示例** — `_collect_usage_patterns()` 从源码中提取调用方式

看起来不少。执行 pyOV 时，首轮 System Prompt + User Prompt 合计约 22 万字符（log 显示 "User Prompt (213835 chars)"），信息量其实是够的。

那问题出在哪？

### 1.2 缺的不是信息量，是"怎么把东西串起来"

pyOV 的 handler 层函数签名是这样的：

```python
def handle_tree(query: dict, body: dict | None, ctx: dict) -> tuple[int, dict]:
```

LLM 通过 API manifest 知道这个签名。但它不知道 `ctx` 里该放什么。

`ctx` 的真实结构来自服务端初始化代码（`run_server.py`）：

```python
store = TripleStore()
store.load_directory(onto_dir)
tbox = TBoxEngine(store)
abox = ABoxEngine(store)
mapper.load_mapping(mapping_file)
ctx = {"store": store, "tbox": tbox, "abox": abox, "mapper": mapper}
```

LLM 看不到这段代码——它不在任何 handler 文件里，也不是 API manifest 的一部分。LLM 只能猜：创造一个 `MagicMock` 充当 `store`，编一个假的 `tbox.get_roots()` 返回值。结果 3 个测试全部在构造 `ctx` 这一步就崩了——不是 assertion 失败，是根本跑不起来。

换句话说，debugloop 给 LLM 的是**散落的零件**（源码、签名、导入路径），但没给它**装配图**。

具体到 pyOV，缺失的关键上下文有三层：

**第一层：fixtures / 运行时构造方式。** `conftest.py` 只做了路径设置，没有提供 `engine_context` fixture。但已有测试 `test_handlers.py` 展示了标准做法——启动真实 HTTP 服务器，加载测试本体数据，通过 HTTP 请求测试每个 handler。LLM 生成的测试试图直接 import handler 函数并手动构造 `ctx`，这条路在 pyOV 的项目架构下根本走不通。

**第二层：已有测试范例。** pyOV 有 22 个已有测试文件，其中 `test_handlers.py` 长达 800+ 行，覆盖了相同 handler 层的所有主要分支。这份代码是天然的 few-shot 范例。debugloop 目前有提取"函数调用方式"的能力，但没有把同模块已有测试代码作为范例注入到 prompt 中。

**第三层：项目分层与数据流。** pyOV 是典型的分层架构——`src/engine/` (数据层) → `src/api/handlers/` (业务层) → `src/api/server.py` (路由层)。handler 不直接持有 store/tbox/abox，它们通过 `ctx` 注入。LLM 从静态代码分析中能推断出分层，但推断不出各层之间的依赖关系和初始化顺序。

### 1.3 在 pyOV 上的具体表现

debugloop 报告显示：

- 基线覆盖率 33.9%（979/2888 分支已覆盖）
- handler 层全部 10 个文件、约 942 个未覆盖分支（`src/api/handlers/` 下 `ontology_handler.py` 等）
- 轮次 1 生成 2 个测试，轮次 2 生成 1 个测试——全部失败
- 失败类型：AssertionError、类型/值错误、未归类错误
- 2 轮后触发 stagnation 终止

值得注意的是，debugloop 对 3 个失败测试都判定为"源码问题"而非"测试生成问题"。但实际上日志没有显示这些测试通过了 `ctx` 构造——它们很可能在 setup 阶段就抛异常了，根本没走到断言。

handler 层的 942 个分支不是无法覆盖——`test_handlers.py` 就覆盖了大量分支。问题在于 LLM 不知道"正确的测试长什么样"，所以每次都从零猜测。

---

## 2. 方案设计："地图"自动提取

核心思路：**在构造 prompt 之前，自动扫描项目的测试 infrastructure，生成一份"项目地图"注入到 LLM 上下文中。**

不要求 LLM 变聪明，而是让它有足够的信息来做出正确的选择。

### 2.1 自动提取的内容和优先级

按收益从高到低排列：

**P0 — conftest.py + 项目级 fixture 提取。** 扫描项目根目录和 `tests/` 下的所有 `conftest.py`，AST 解析提取 fixture 定义（函数名、返回值类型、依赖的其他 fixture）。对于 pyOV 这种情况（conftest.py 只有路径设置），还需要提取 `run_server.py` 中的引擎初始化模式。

输出示例：
```markdown
## 项目运行时上下文

### Fixtures（来自 conftest.py）
- (无可用 fixture — conftest.py 仅做路径设置)

### 引擎初始化（来自 run_server.py 扫描）
ctx 字典结构：
  - "store": TripleStore 实例（需先 load_directory()）
  - "tbox": TBoxEngine(store)
  - "abox": ABoxEngine(store)
  - "mapper": MappingEngine 实例
  - "reasoner": ReasonerEngine(store)
  - "searcher": SearcherEngine(store, reasoner=reasoner)

### 推荐测试构造方式
测试 handler 有两种路径：
1. 启动服务器 → HTTP 请求（参考 tests/test_handlers.py）
2. 手动构造 ctx → 直接调用 handler 函数
```

这个信息量不大，但对 LLM 来说是决定性的——它不再需要猜测 `ctx` 的结构。

**P1 — 同模块已有测试范例。** 对于每个未覆盖分支所在的模块（如 `src/api/handlers/ontology_handler.py`），在 `tests/` 下找到对应的测试文件（如 `tests/test_handlers.py`），取最近 1-2 个测试类/函数作为 few-shot 范例注入。范例展示的是：正确的 import 方式、如何构造测试数据、如何调用被测函数。

**P2 — README.md 提取。** 从项目 README 中提取 Quick Start 命令、架构说明段落、测试运行方式。这些信息帮助 LLM 理解项目的整体运行方式。

**P3 — 目录结构分层推断。** 扫描 `src/` 下的目录结构，识别典型分层模式（如 `api/handlers/` → handler 层、`engine/` → 数据层）。生成分层描述和数据流示意图。

**P4 — `docs/architecture.md` 等设计文档。** debugloop 已有 `_load_design_docs()` 函数读取 `docs/`，这一步只需要加强——提取关键段落而非简单截断前 3000 字符。

### 2.2 注入方式

所有提取结果注入到 `build_user_prompt()` 的 prompt 中，新增一个 `project_context` section，放在"未覆盖分支"之前，API manifest 之后。

```
## 项目上下文（自动提取）

[P0: fixtures & 运行时上下文]
[P1: 测试范例]
[P2: README 关键段落]
[P3: 目录分层推断]
```

不需要改 prompt 的 System Prompt 部分——规则、断言体系、Mock 红线保持不变。只需要让 User Prompt 包含足够的信息即可。

### 2.3 与现有注入能力的关系

debugloop 目前已经有了一些上下文注入能力：

| 现有能力 | 覆盖范围 | 缺口 |
|----------|---------|------|
| API 签名清单 (`build_api_manifest`) | 所有模块的函数/类签名 | 不包含模块间依赖关系 |
| 项目导入路径 (`_discover_project_imports`) | 可导入符号 | 不包含运行时构造方式 |
| 设计文档 (`_load_design_docs`) | docs/*.md | 只做简单截断，不提取关键信息 |
| 函数调用示例 (`_collect_usage_patterns`) | 源码中的调用方式 | 不包含测试文件中的用法 |
| 函数源码注入 (V43) | 未覆盖分支所在函数 | 不含外部依赖上下文 |

"地图"方案不是推倒重来，而是补上最关键的一块拼图——**让 LLM 知道被测代码的运行时依赖关系和项目中已有的正确做法。**

---

## 3. 可行性分析

### 3.1 技术可行性

每个子任务都有成熟的实现路径：

- **conftest.py AST 解析提取 fixture 签名**：Python 标准库 `ast` 即可完成。提取 `@pytest.fixture` 装饰的函数，记录函数名、参数（依赖的其他 fixture）、返回值类型注解。对于非 fixture 但包含关键初始化逻辑的函数（如 `run_server.py` 中的引擎初始化），用启发式规则识别（import 了多个 engine 类、构造了 dict 或类似结构并包含 "store"/"tbox" 等 key）。

- **已有测试范例的匹配和提取**：通过文件名规则匹配（`test_handlers.py` ← `handlers/ontology_handler.py`）或启发式匹配（测试文件中 import 了目标模块）。提取完整测试类或函数作为范例。复杂度低，主要工作是匹配算法的打磨。

- **目录结构分层推断**：纯文件系统操作 + 简单规则。检测 `src/api/handlers/` → handler 层、`src/engine/` → 数据层、`src/cli/` → 命令行层。已有项目（如 React 项目按 components/hooks/utils 分层）也能覆盖。

- **README.md 关键段落提取**：正则匹配 `## 快速开始`、`## 架构` 等 section，提取 QDD 示例块和架构说明文字。不需要 NLP，正则 + Markdown parser 足够。

整体工作量估计 3-5 个工作日。技术栈全部在 debugloop 已有的 Python 能力范围之内，不需要引入新依赖。

### 3.2 效果预估

以 pyOV handler 层 942 个分支为例做推演：

**方案前（当前状态）：** LLM 收到 handler 源码 + API 签名 + 导入路径。它尝试直接构造 `ctx` dict，但由于不知道 TripleStore 需要什么参数、TBoxEngine 的初始化顺序、以及这些对象之间的依赖关系，生成的测试在 setup 阶段就崩溃。通过率 0/3，覆盖率提升 0%。

**方案后（保守估计）：** LLM 收到 handler 源码 + API 签名 + 导入路径 + 运行时上下文（ctx 结构） + `test_handlers.py` 范例 2 个。

LLM 看到范例后有两种合理选择：
1. **走 HTTP 路径**（推荐）：启动服务器 → 构造 HTTP 请求 → 验证响应，与已有测试风格一致
2. **走直接调用路径**：创建一个真实的 TripleStore + TBoxEngine 实例，加载测试本体数据，构造正确的 ctx dict

两种路径 LLM 目前都做不到，因为缺信息。有了上下文后，第一种路径可以直接照抄范例模式，第二种路径也有了构造 ctx 的模板。

预期效果：
- handler 层单轮测试通过率从 0/3 提升到至少 **1/3 以上**
- 如果 LLM 走 HTTP 路径，可以覆盖 handler 的主要正常路径分支（error 分支则需要单独构造异常输入）
- 保守估计，2-3 轮迭代后 handler 层覆盖率可以从接近 0% 提升到 **30-50%**

这不能解决所有问题——handler 的异常路径、边界条件仍然需要 LLM 深度理解业务逻辑才能触发。但它解决了最大的瓶颈：**连跑都跑不起来的测试，谈覆盖率为时尚早。**

### 3.3 风险

**Prompt 长度膨胀。** 注入 conftest 内容 + 测试范例 + README 段落，User Prompt 可能从当前的 ~21 万字符增加到 25-30 万字符。一方面增加 token 消耗（每次多 ~¥0.5-1），另一方面可能超出部分模型的上下文窗口。缓解措施：每个 section 做长度限制——fixture 描述限制在 500 字符内，测试范例限制 1-2 个且截断到 100 行，README 限制 500 字符。

**范例选择的准确度。** 自动匹配测试文件和源码模块的算法可能出错——比如把 `test_integration.py` 错误匹配给 `ontology_handler.py`。错误的范例比没有范例更糟，因为它会误导 LLM。缓解措施：优先使用文件名前缀匹配（`test_handlers.py` → `handlers/`），再使用内容匹配（测试文件 import 了目标模块），最后回退到不注入。

**conftest 风格差异大。** 不同项目的 conftest 写法天差地别。有的项目用 `@pytest.fixture` 定义标准 fixture，有的项目在 conftest 里做 path hack，有的项目根本没有 conftest。AST 解析需要足够鲁棒。缓解措施：对于无 fixture 的情况，回退到扫描 `run_server.py`、`main.py`、`app.py` 等入口文件中的初始化逻辑。

**幻觉风险。** LLM 可能会把范例中的具体数值（如测试本体 URI）直接复制到新测试中，而不理解这些值需要根据目标分支调整。缓解措施：在 prompt 中明确标注"以下范例仅供风格参考，请根据目标未覆盖分支调整测试数据和断言"。

### 3.4 回退机制：人类手动覆盖

自动提取不可能覆盖所有项目模式。方案设计一个手动覆盖入口：

**在项目根目录放置 `dl-map.md` 文件。** 如果该文件存在，debugloop 跳过所有自动提取，直接读取文件内容注入到 prompt 中。

`dl-map.md` 的结构：
```markdown
# debugloop 项目上下文（手动编写）

## 运行时上下文
ctx = {"store": TripleStore(...), "tbox": TBoxEngine(store), ...}

## 正确测试范例
参见 tests/test_handlers.py 的 test_tree_normal 方法

## 注意事项
- 不要直接 mock TripleStore，它的初始化涉及 rdflib Graph
- 测试数据在 ontologies/testonto/ 下
```

这样，自动提取覆盖 80% 的常见场景，剩下 20% 的特殊项目由人手动补充。自动提取和手动覆盖不冲突——`dl-map.md` 存在则优先使用，不存在则自动提取。

---

## 4. 实施路径

### Phase 1：conftest.py 解析 + fixtures 注入（最小可行改动）

**目标：** 让 LLM 知道"怎么把对象串起来"。

**工作内容：**
- 新增 `_build_project_context()` 函数，扫描项目 conftest.py 并提取 fixture 定义
- 新增 `_extract_engine_context()` 函数，扫描入口文件（run_server.py / app.py / main.py）提取初始化模式
- 在 `build_user_prompt()` 中增加 `project_context` 参数，注入提取结果
- 对 pyOV 做回归测试，验证 handler 层测试通过率提升

**工期：** 1-2 天

**收益：** 解决"LLM 不知道 ctx 里放什么"的核心问题。对大多数需要运行时上下文的项目（FastAPI/Flask handler、Django view 等）都有效。

### Phase 2：同模块测试范例注入

**目标：** 让 LLM 有成功的范例可以参考。

**工作内容：**
- 新增 `_find_related_tests()` 函数，按模块名匹配已有测试文件
- 新增 `_extract_test_examples()` 函数，从匹配到的测试文件中取 1-2 个完整测试函数/类
- 注入到 `project_context` section 中，附带"仅供风格参考"提示
- 控制范例长度，避免 prompt 过度膨胀

**工期：** 1 天

**收益：** 让 LLM 在"已知正确的模式"上叠加，而非从零猜测。与 Phase 1 叠加后，pyOV handler 层通过率预期可达 30-50%。

### Phase 3：目录结构分层推断

**目标：** 让 LLM 理解项目架构，做出更合理的测试设计。

**工作内容：**
- 新增 `_infer_project_layers()` 函数，扫描目录结构并生成分层描述
- 支持常见分层模式：MVC（api/handlers + engine + templates）、DDD（domain + application + infrastructure）、微服务（services/ + shared/）
- 生成的描述附带数据流示意

**工期：** 1 天

**收益：** 锦上添花——对于分层清晰的项目有帮助，但不如 Phase 1 和 2 关键。

### Phase 4：`dl-map.md` 人类覆盖机制

**目标：** 提供兜底方案，覆盖自动提取无法处理的场景。

**工作内容：**
- 在 `_build_project_context()` 中增加 `dl-map.md` 检测逻辑
- 文件存在 → 直接读取注入，跳过自动提取
- 文件不存在 → 执行 Phase 1-3 的自动提取
- 输出提示：建议人类为复杂项目编写 `dl-map.md`

**工期：** 0.5 天

**收益：** 长尾覆盖。少数复杂项目需要人工介入，这个机制让它们不至于被放弃。

---

## 5. 参考数据

以下数据来自 debugloop 在 pyOV 项目上的实际运行记录（2026-06-27 18:30-18:38）：

| 指标 | 数值 |
|------|------|
| 基线覆盖率 | 33.9%（979/2888 分支） |
| 目标覆盖率 | 85% |
| 迭代轮次 | 2 |
| 终止原因 | 覆盖率停滞 |
| 生成测试数 | 3 个（轮次 1: 2 个，轮次 2: 1 个） |
| 测试通过数 | 0 |
| 失败类型 | AssertionError、类型/值错误、未归类错误 |
| handler 层未覆盖分支 | ~942 个（10 个 handler 文件） |
| 最大 prompt 大小 | ~214,000 字符（User Prompt） |
| 总 token 消耗 | ~¥3.88（2 轮） |

debugloop 报告文件：`pyOV/dl-test/reports/debugloop-report.json`

整体来看，debugloop 在 pyOV 上的问题不是工具设计缺陷，而是一个**信息不对称**问题：LLM 有足够的能力写出正确的测试（pyOV 的已有测试证明了这点），但它缺了一份"项目地图"来理解数据怎么流动、对象怎么构造。

这份方案要解决的，就是填上这个缺口。
