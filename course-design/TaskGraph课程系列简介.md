## 修正后的整体课程设计压缩（基于后续定位修正）

### 课程总定位
**《从 Claude Code 到 Task Graph：重建 AI Agent 的任务态上下文工程》**

不再定位为单纯解读源码或还原 AI 思考，而是：**从 Claude Code 暴露的 Agent 工程线索出发，学习上下文管理、压缩、功能演化，并原创实验系统 Task Graph——任务态上下文管理管线**。

**核心主线修正**：Claude Code 展示工业级应对上下文爆炸的手段；Task Graph 不是通用压缩器，而是**任务态依赖索引与上下文结构化机制**。最终形成管线：
```
用户对话 → Graph Slice Builder → Graph Extractor → patch*.json（不可变事件日志）
→ reconcile_patch.py（事后校验）→ apply patch → graph_state.json（编译态）
→ graph_lint.py（全局不变量检查）→ Context Builder → context_pack.json / context_prompt.md → 主 LLM
```
扩展管线：
- **Memory**：TaskGraph → Memory Extractor → Memory Store → 召回 → 新 TaskGraph → Context Pack
- **多 Agent**：Coordinator 基于 TaskGraph 生成 Worker Context Pack → Worker 返回 Result + Patch → Coordinator 合并
- **证据**：Tool / File / Test → TaskGraph → Context Pack / Debug Report

---

### 课程结构（10 + 1 讲，含新增的第 4.5 讲）

#### 第0讲：导论——从 Claude Code 到 Task Graph
- 44 个 feature flags 是理解系统演化的入口，上下文管理是核心战场。
- 引出课程双线：Claude Code 的工程方法 + Task Graph 的实验性回应。
- 核心结论：Claude Code 暴露的是 Agent 工程方法论，而非单个功能。
- 产物：课程总地图，Task Graph 初步概念图。

#### 第1讲：门控体系——复杂 Agent 如何控制功能演化
- 六层门控（编译期、环境变量、远程实验、配置文件、API 能力、Kill Switch）。
- 以 COORDINATOR_MODE 等为例追踪真实门控链路。
- 迁移设计：为 Task Graph 原型设计五层门控（建图、调试、GC、可视化、实验特性）。
- 结论：门控是管理演化复杂性的架构能力。

#### 第2讲：上下文工程——Claude Code 如何在有限窗口中持续工作
- 上下文预算模型（固定开销 + 可变开销 + 净空间）。
- 五层防御体系：源头控制 → 折叠压缩（Context Collapse）→ 微压缩（MicroCompact）→ 全量压缩（Auto Compact）→ 应急压缩（Reactive Compact）。
- 摘要强结构化（9 个固定栏目）+ `<analysis>` 草稿区 + 禁止工具调用双保险。
- 对 Task Graph 启示：梯度响应、摘要作为消息、断路器、按语义单元分组；图结构可实现比无差别摘要更智能的压缩。

#### 第3讲：从 Micrograd 到 Task Graph——上下文不是文本，而是依赖图
- Micrograd 的 `Value` 类（`_prev`, `_op`）启发依赖追踪。
- Task Graph 定位修正：**不是还原模型内部思维，而是记录任务中外部可验证依赖**。
- 节点类型：UserGoal, Constraint, Fact, ToolResult, Decision, FileArtifact, OpenTask。
- 核心价值：回答"哪些旧信息不能丢""前提变化后哪些结论失效""工具错误后哪些答案受影响"。

#### 第4讲：写入侧——从对话到 Graph Patch，再到 TaskGraph
- **本讲定位**：用户对话 → Graph Extractor → Graph Patch → TaskGraph，回答"任务图怎么长出来"。
- 任务级图 vs 轮次级图：一个任务一张图，每轮只做增量 Patch（new / updated / superseded）。
- 启动时机：手动（如 `/task start`），后续可半自动 / 全自动。
- 典型操作：通过 Graph Extractor Prompt 让 LLM（旁路小模型）输出结构化 Patch，Python 代码合并成 TaskGraph。
- 最小可运行代码：Node, Edge, TaskGraph, GraphUpdater, Deduplicator, to_mermaid。
- 核心原则：不删除被推翻节点，用 `superseded` 留下演化轨迹。

#### 第4.5讲：长程运行补强——Context Pack、Reconciliation 与 Graph Lint
- **本课定位**：基础设施补强课。解决完整 `graph_state.json` 随轮次膨胀导致 Extractor 上下文超限、准确性下降问题。**不补则系统无法长程运行**。
- 核心改造：
  - **Graph Slice Builder**：不再喂全图，只生成本轮相关的 `extractor_context_pack.json`（有限切片）。
  - **entity_ref / state**：节点增加结构化实体引用和状态枚举，将冲突判定降级为机械精确匹配。
  - **reconcile_patch.py**：事后校验，用完整图复核 patch 自洽性（缺 superseded / invalidates 则报错）。
  - **graph_lint.py**：应用后校验全局不变量（孤立节点、互斥 state、relation 合法性等）。
  - **失败处理**：有界重试（上限 N=2~3 次）+ 失败隔离（超限则拒绝合并，隔离到 `quarantine/`），**不污染主图**。
- 核心原则：Slice Builder 只负责省 token，不负责正确性；正确性由全图级机械校验 + 失败隔离保证。

#### 第5讲：读取侧——从 TaskGraph 到 Context Pack
- **本讲定位修正**：不是压缩，而是 **TaskGraph → Context Pack → 主 LLM**，回答"任务图怎么喂回去"。
- Context Pack 是结构化任务上下文（current_focus, goals, active_constraints, current_decisions, relevant_facts, open_tasks, superseded_history），不是图 dump。
- 从当前焦点（如最新 active OpenTask）向上游追踪依赖边（`depends_on` / `implements` / `serves` / `refines`），同时收集被 `invalidates` 边标记的旧方案。
- 最小代码实现 `build_context_pack` + `context_pack_to_prompt`，注入主 LLM。
- 核心价值：让主 LLM 明确知道当前有效约束、已推翻方案及原因，避免重蹈覆辙。

#### 第6讲：长期记忆——从 TaskGraph 到 Memory Store
- **本讲定位修正**：不是"压缩升级版"，而是跨任务、跨会话的**长期任务记忆沉淀与召回**。
- 流程：任务阶段结束 → Memory Extractor 抽取长期记忆 → Memory Store；新任务时召回 → 生成 MemoryReferenceNode 进入新 TaskGraph → Context Pack。
- 沉淀类型：GoalMemory, ConstraintMemory, PreferenceMemory, DecisionMemory, OpenTaskMemory, ErrorMemory, ArtifactMemory（可选）。
- 不沉淀：临时推理、大段工具输出、已推翻方案、闲聊。
- 生命周期：create, update, merge, invalidate, archive, delete。失效标记而非删除，留痕供后续参考。
- 召回后通过 `memories_to_seed_nodes` 接入 TaskGraph，不走直接塞 prompt。

#### 第7讲：多 Agent 协作——Worker Context Pack 与 Patch 合并
- **本讲定位**：解决多 Agent 时 TaskGraph 的一致性、边界性和可验证性。
- 核心架构：Coordinator 基于 TaskGraph 为每个 Worker 生成 **Worker Context Pack**（子任务上下文）；Worker 只返回 **Result + Graph Patch**（不直接改图）；Coordinator 合并 Patch、处理冲突；Verifier 独立验证关键 Claim + Evidence，返回 Verification Patch。
- 三种包：Worker Context Pack, Worker Result Package, Verification Pack。
- Demo 延续 Minecraft：拆分 3 个 Worker → 生成 Context Pack → 返回 Patch → 合并时发现客户端命令与无 OP 权限冲突 → Verifier 验证 RCON → 更新 TaskGraph → 重新生成主 LLM Context Pack。
- 核心原则：TaskGraph 单写入，Worker 多读取；广播式上下文爆炸被消除。

#### 第8讲：工具、文件与测试——让外部证据进入 TaskGraph
- **本讲定位**（中等调整）：工具调用、文件产物、测试结果作为可追踪、可压缩、可失效的上下文证据。
- 四类核心节点：ToolCallNode, ToolResultNode, FileArtifactNode（含版本哈希）, TestResultNode。
- 大结果写盘 + 预览引用（preview + full_ref），连接第 5 讲压缩（可丢 preview，不可丢被依赖的 full_ref）。
- 失效传播简化版：当工具结果错误或文件变化，标记 `tainted` 向下游传播。
- Demo：根据 config.yaml 写代码 → 测试 → 修改 config.yaml → 下游节点变 tainted / suspect / invalid。
- 核心结论：信息有来源，依赖有方向，文件有版本，结果能回查。

#### 第9讲：可观测性——Task Context Debugger
- **本讲定位重写**：不是"AI Reasoning Debugger"，而是 **TaskGraph 管线的工程调试器**。
- 核心问题：系统输出错了，是 Extractor 抽错、Graph 合并错、Context Pack 漏、Memory 召回错，还是证据失效？
- 六类可观测对象：Patch, TaskGraph Snapshot, Context Pack, Memory Recall Report, Evidence Index, Event Timeline。每类必须包含 `reason` 字段。
- **本讲最重要部分**：Context Pack Trace——为 Pack 中每个条目增加 `included_reason` / `excluded_reason`，解释为何入选 / 排除。
- 产出静态 **Task Context Debug Report**（含 conversation.md, patches.json, task_graph.json, context_pack.md, memory_recall.json, evidence_index.json, timeline.json, graph.mmd）。
- 实时 UI 作为选做扩展。

#### 第10讲：总结与路线图——完整任务态上下文系统闭环
- **本讲定位重写**：闭环前 9 讲 + 第 4.5 讲形成完整工程管线；明确 Task Graph"是什么、不是什么"；给出后续路线。
- **明确定位**：Task Graph 是**任务态上下文管理机制**，不是模型内部推理还原，不是通用聊天压缩器，不是 Compact 替代品，不是万能长期记忆，不是可上线的工业系统。
- **最终 Demo（Minecraft 全流程）**：
  用户提出任务 → Extractor 输出 Patch（目标 / 偏好 / 事实）→ 补充约束（一个键、无 OP 权限）→ Patch 新增 Constraint，标记旧客户端方案 superseded，新增服务端 RCON 决策 → Context Builder 生成 Context Pack（目标、有效约束、当前方案、已推翻方案）→ Memory 沉淀 → 新会话召回记忆 → 工具证据入图（读取配置、测试 RCON）→ Debug Report 解释为何建议 RCON 而非客户端命令。
- **后续路线图**：
  - **工程**：封装 Python 包（extractor / graph / context_builder / memory / coordinator / evidence / debug_report / examples）。
  - **产品**：做成"Task Context Debugger for AI Agents"，卖点为主 LLM 可见上下文、漏掉约束、旧方案被推翻、工具证据支撑、多 Agent 合并。
  - **研究**：抽取准确率、Context Pack 对长任务完成率提升、Memory 过期冲突、多 Agent Patch 一致性。
  - **教学**：可打包成 AI Agent 工程课 / 上下文管理专题课。
- **最终收束**：课程真正主线不是 Claude Code 的某个 flag，而是**复杂 Agent 系统如何管理任务状态**。Claude Code 展示了工业手段，Task Graph 尝试回答：若显式记录目标、约束、决策、证据和被推翻方案，Agent 能否更稳定地继续工作。

---

### 核心工程管线（第 4–9 讲横切原则）

1. **任务态启动**：无任务不建图，只服务目标明确、约束持续、方案演化的任务型对话。
2. **写入 / 读取分离**：第 4 讲（图怎么长出来），第 5 讲（图怎么喂回主 LLM）。
3. **Patch 是源头，graph_state 是编译产物**：`patch*.json` 不可变，`graph_state.json` 只能由 patch 重编译，禁止手改。
4. **Extractor 读 Graph Slice，不读完整图**：完整图是数据库，不是 prompt。
5. **Slice 不承担正确性保证**：Slice 只负责尽量召回；正确性由 `entity_ref`、`state`、reconciliation、lint 等机械校验兜底。
6. **语义判断归 Extractor，机械合并归 Applier**：`superseded_nodes`、`invalidates` 由 Extractor 输出；Applier 只合并，不做语义推理。
7. **Context Builder 只读，不回写图**：可检测冲突、降级展示，但不修改 TaskGraph。
8. **手动流程是自动化流程的可观察版本**：手动复制文件、运行脚本是自动化管线的透明形态，不是临时旁路。
9. **外部证据节点化**：工具调用、工具结果、文件版本、测试结果都应进入图，成为可追踪、可失效、可回查的证据节点。
10. **调试对象是管线中间产物**：第 9 讲调试 Patch、TaskGraph、Context Pack、Memory Recall、Evidence、Timeline，不解释模型内心。

---

### 技术产出（项目结构）
```
task_graph_context_manager/
├── scripts/
│   ├── build_graph_slice.py       # 从完整图 + 本轮对话生成本轮切片
│   ├── build_extractor_prompt.py  # 组装 system + user prompt（支持 single / split / api-json 模式）
│   ├── apply_patch.py             # 机械合并 patch 到 graph_state
│   ├── reconcile_patch.py         # 事后校验：patch 与全图自洽性（entity_ref 精确匹配）
│   ├── graph_lint.py              # 全局不变量检查（孤立节点、互斥 state、relation 合法性等）
│   ├── quarantine_patch.py        # 失败隔离：超限 patch 写入 quarantine/
│   └── context_pack_builder.py    # 从 TaskGraph 生成 Context Pack / context_prompt.md
├── prompts/
│   ├── extractor_system.md        # 固定规则（可缓存前缀）
│   └── turns_promts/              # 每轮组装后的完整 prompt（per-turn）
├── graph_slices/                  # 本轮相关图切片（extractor_context_pack.json）
├── patches/                       # 不可变事件日志（patch_*.json）
├── original_dialogs/              # 原始对话文本（turn_*.md）
├── run/                           # 运行时产物（graph_state.turn_*.json）
├── quarantine/                    # 隔离的失败 patch
├── reports/                       # 验证报告与调试报告
├── tests/
│   ├── fixtures/                  # 测试夹具
│   └── smoke/                     # 冒烟测试
└── graph_state.json               # 编译态产物（当前已 apply 的所有 patch 合并视图）
```

---

### 核心价值总结（修正后）

1. **有热点入口**：Claude Code 的 flags 和上下文机制。
2. **有工程深度**：门控、上下文压缩、预算、工具、记忆、多 Agent、可观测性。
3. **有原创产出**：Task Graph 作为任务态上下文管理管线，强调 **写入（Patch → TaskGraph）→ 读取（Context Pack）→ 长程运行补强（Slice + Reconciliation + Lint）→ 长期记忆（Memory）→ 多 Agent 边界 → 证据追踪 → 调试报告** 的闭环。
4. **最重要的路线修正（已落地）**：
   - 原"依赖感知压缩" → 改为"任务图写入与读取"，压缩为副产物。
   - 原"AI 推理调试器" → 改为"TaskGraph 管线调试器"。
   - 原"通用上下文压缩" → 改为"任务态上下文管理，无任务不建图"。
   - **新增"长程运行补强"（第 4.5 讲）**：通过 Graph Slice + `entity_ref` 精确匹配 + Reconciliation + Lint + 失败隔离，使 Task Graph 可长程运行而不超限、不失真。
