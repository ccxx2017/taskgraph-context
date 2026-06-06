
## 课程名称

**从 Claude Code 到 Task Graph：重建 AI Agent 的任务态上下文工程**

## 课程定位

本课程从 Claude Code 的 feature flags 和上下文工程机制出发，逐步提出一套面向任务型对话的 **Task Graph / TaskGraph 上下文管理方案**。

Task Graph 的定位是：

- 不是通用聊天压缩器；
- 不是 Claude Code Compact 的替代品；
- 不是模型内部推理还原；
- 不是万能长期记忆系统；
- 而是：在进入任务态后，显式记录目标、约束、事实、决策、工具结果、文件产物、记忆引用和被推翻方案，并将其转换为主 LLM 可使用的结构化 Context Pack。

---

## 核心工程管线

```text
用户对话
  ↓
Graph Slice Builder
  ↓
Graph Extractor
  ↓
patch*.json 不可变事件日志
  ↓
Patch Applier 机械合并
  ↓
graph_state.json 编译态结果
  ↓
Context Builder
  ↓
context_pack.json / context_prompt.md
  ↓
主 LLM
```

扩展管线：

```text
TaskGraph → Memory Extractor → Memory Store
Memory Store → Recall → MemoryReferenceNode → 新 TaskGraph → Context Pack

Worker Context Pack → Worker Result + Graph Patch → Coordinator Merge
Tool/File/Test Evidence → TaskGraph → Context Pack / Debug Report
```

---

## 第4–9讲横切原则

1. **任务态启动**  
   无任务不建图。Task Graph 只服务目标明确、约束持续、方案会演化的任务型对话。

2. **写入侧 / 读取侧分离**  
   第4讲解决“任务图怎么长出来”；第5讲解决“任务图怎么喂回主 LLM”。

3. **Patch 是源头，graph_state 是编译产物**  
   `patch*.json` 是不可变事件日志；`graph_state.json` 只能由 patch 重新编译生成，禁止手改。

4. **Extractor 读 Graph Slice，不读完整图**  
   完整 TaskGraph 是数据库，不是 prompt。Extractor 只读取本轮相关切片。

5. **Graph Slice 不承担正确性保证**  
   Slice 只负责尽量召回相关信息；正确性由 `entity_ref`、`state`、reconciliation、lint 等机械校验兜底。

6. **语义判断归 Extractor，机械合并归 Applier**  
   `superseded_nodes`、`invalidates` 等语义关系由 Extractor 输出；Applier 只合并，不做语义推理。

7. **Context Builder 只读，不回写图**  
   它可以检测冲突、降级展示、生成 `superseded_history`，但不修改 TaskGraph。

8. **手动流程是自动化流程的可观察版本**  
   手动复制文件、运行脚本、检查输出，不是临时旁路，而是未来自动化管线的透明形态。

9. **外部证据必须节点化**  
   工具调用、工具结果、文件版本、测试结果都应进入图，成为可追踪、可失效、可回查的证据节点。

10. **调试对象是管线中间产物，不是模型内心**  
   第9讲调试 Patch、TaskGraph、Context Pack、Memory Recall、Evidence、Timeline，而不是解释模型隐式思维。

---

# 课程大纲

## 第0讲：导论——从 Claude Code 到 Task Graph

**核心问题**：复杂 Agent 系统如何在有限上下文中保持任务连续性？

主要内容：

- Claude Code feature flags 暴露出的工程线索；
- 为什么上下文管理是 Agent 系统的核心问题；
- 从 Micrograd 的计算图想到任务依赖图；
- 课程主线：从“观察 Claude Code”到“构建自己的任务态上下文系统”。

产物：

- 课程总地图；
- Task Graph 的初步概念图。

---

## 第1讲：门控体系——复杂 Agent 如何控制功能演化

**核心问题**：复杂系统如何让实验功能、安全性和稳定性共存？

主要内容：

- Claude Code 的多层门控：
  - 编译期 feature flag；
  - 环境变量；
  - 用户类型；
  - 远程实验；
  - 配置文件；
  - API 能力；
  - Kill Switch。
- 门控体系对 Task Graph 原型的启发；
- 为 TaskGraph 系统设计功能开关。

产物：

- Task Graph 原型门控表；
- 功能开关分层设计。

---

## 第2讲：上下文工程——Claude Code 如何管理有限窗口

**核心问题**：Agent 越能干，为什么越容易耗尽上下文？

主要内容：

- Claude Code 的上下文防御体系：
  - 工具输出截断；
  - Context Collapse；
  - MicroCompact；
  - Auto Compact；
  - Reactive Compact。
- Compact Prompt 的结构化设计：
  - 9个固定栏目；
  - no-tools 双保险；
  - `<analysis>` 作为临时草稿；
  - summary 作为 continuation payload。
- 现有压缩策略的局限：
  - 主要围绕 token、时间顺序和消息结构；
  - 缺少对任务依赖关系的显式建模。

产物：

- Claude Code 上下文工程流程图；
- Task Graph 的问题入口。

---

## 第3讲：从 Micrograd 到 Task Graph

**核心问题**：能否像计算图追踪数值依赖一样，追踪任务上下文依赖？

主要内容：

- Micrograd 中 `_prev`、`_op`、trace 的思想；
- 从计算节点迁移到任务上下文节点；
- Task Graph 追踪的是外部可记录任务依赖，不是模型内部思维；
- 基础节点类型：
  - UserGoal；
  - Constraint；
  - Fact；
  - Decision；
  - OpenTask；
  - ToolResult；
  - FileArtifact。
- 基础关系：
  - refines；
  - depends_on；
  - implements；
  - serves；
  - derived_from；
  - supports；
  - invalidates。

产物：

- Task Graph 基础数据模型；
- 简单任务依赖图示例。

---

## 第4讲：写入侧——从对话到 Graph Patch，再到 TaskGraph

**核心问题**：任务图如何从多轮对话中持续长出来？

主要内容：

- 一个任务一张 TaskGraph；
- 每轮对话生成一次 Graph Patch；
- Patch 包含：
  - new_nodes；
  - updated_nodes；
  - superseded_nodes；
  - new_edges。
- 旁路 Graph Extractor 架构；
- Extractor 不读取完整图，而读取 Graph Slice；
- Graph Slice 包含：
  - root_goals；
  - standing_constraints；
  - active_open_tasks；
  - recent_nodes；
  - relevant_nodes；
  - conflict_candidates；
  - next_node_id。
- 引入 `entity_ref` 与 `state`；
- 事实一致性反向检查：
  - 同一实体状态变化；
  - 旧事实失效；
  - 旧决策被替代；
  - 阻塞状态被解除；
  - 文件路径或接口定义更新。
- Patch Applier 只做机械合并；
- `patch*.json → graph_state.json` 的编译流程。

产物：

- Graph Extractor Prompt；
- 示例 patch 文件；
- patch applier；
- 编译生成的 `graph_state.json`。

---

## 第5讲：读取侧——从 TaskGraph 到 Context Pack

**核心问题**：TaskGraph 如何变成主 LLM 能直接使用的任务上下文？

主要内容：

- 为什么不能直接把整张图塞进 prompt；
- Context Pack 的作用：
  - 当前目标；
  - 有效约束；
  - 当前决策；
  - 相关事实；
  - 未完成任务；
  - 已推翻方案；
  - 证据引用。
- 从当前焦点读取子图：
  - 焦点追踪；
  - 依赖追踪；
  - 邻居扩展；
  - 冲突扫描；
  - active / superseded 区分。
- Context Builder 只读，不回写 TaskGraph；
- 输出：
  - `context_pack.json`；
  - `context_prompt.md`。
- included_reason / excluded_reason 的初步引入。

产物：

- `context_pack_builder.py`；
- `context_pack.json`；
- `context_prompt.md`；
- 主 LLM 可复制使用的任务上下文片段。

---

## 第6讲：长期记忆——从 TaskGraph 到 Memory Store

**核心问题**：任务结束后，哪些信息值得长期保留，并在新任务中召回？

主要内容：

- TaskGraph、Context Pack、Memory Store 的区别；
- 适合沉淀为记忆的内容：
  - 长期目标；
  - 稳定约束；
  - 用户偏好；
  - 有效决策；
  - 未完成任务；
  - 踩坑记录；
  - 重要文件或产物引用。
- 不应沉淀的内容：
  - 临时推理；
  - 大段工具输出；
  - 已失效方案；
  - 闲聊；
  - 不确定猜测。
- Memory 生命周期：
  - create；
  - update；
  - merge；
  - invalidate；
  - archive。
- 召回路径：
  - Memory Store；
  - Recall；
  - MemoryReferenceNode；
  - 新 TaskGraph；
  - Context Pack。

产物：

- Memory Extractor Prompt；
- Memory Store 简化实现；
- Memory Recall 示例；
- MemoryReferenceNode 注入 TaskGraph 示例。

---

## 第7讲：多 Agent 协作——Worker Context Pack 与 Patch 合并

**核心问题**：多个 Agent 协作时，如何避免上下文广播、冲突放大和图状态失控？

主要内容：

- Coordinator 单点调度；
- TaskGraph 单写入、多读取；
- Worker 不直接修改 TaskGraph；
- Coordinator 为 Worker 生成 Worker Context Pack；
- Worker 返回：
  - result；
  - claims；
  - graph_patch。
- Patch 统一合并；
- 冲突检测：
  - 身份冲突；
  - 内容冲突；
  - 依赖冲突；
  - 状态冲突。
- Verifier 只看 Claim + Evidence；
- Memory 由 Coordinator 召回，不直接暴露给 Worker。

产物：

- Worker Context Pack 模板；
- Worker Result Package 模板；
- Verification Pack 模板；
- 多 Agent Patch 合并流程图。

---

## 第8讲：工具、文件与测试——让外部证据进入 TaskGraph

**核心问题**：工具结果、文件修改和测试结果如何成为可追踪的任务证据？

主要内容：

- 四类证据节点：
  - ToolCallNode；
  - ToolResultNode；
  - FileArtifactNode；
  - TestResultNode。
- 大结果处理：
  - preview；
  - full_ref；
  - 外部文件或对象存储；
  - Context Pack 中只保留必要引用。
- 文件版本追踪：
  - path；
  - version；
  - hash；
  - produced_by；
  - depends_on。
- 测试结果追踪：
  - command；
  - passed / failed；
  - output_ref；
  - related_file。
- 失效传播：
  - 工具结果错误；
  - 文件版本变化；
  - 测试结果过期；
  - 下游节点标记 suspect / invalid。
- 核心原则：
  - 信息有来源；
  - 依赖有方向；
  - 文件有版本；
  - 结果能回查。

产物：

- Evidence Node 数据结构；
- 工具结果入图示例；
- 文件版本依赖链；
- 简化 taint / invalidation 示例。

---

## 第9讲：可观测性——Task Context Debugger

**核心问题**：系统输出错了，如何判断问题出在抽取、合并、读取、召回、证据，还是图漂移？

主要内容：

- 调试对象：
  - patches；
  - task_graph；
  - context_pack；
  - memory_recall；
  - evidence_index；
  - timeline；
  - graph visualization。
- Task Context Debug Report：
  - `patches.json`；
  - `task_graph.json`；
  - `context_pack.md`；
  - `memory_recall.json`；
  - `evidence_index.json`；
  - `timeline.json`；
  - `graph.mmd`。
- Context Pack Trace：
  - 每条信息为什么进入；
  - 每条信息为什么排除；
  - 哪些旧方案被推翻；
  - 哪些证据支撑当前方案。
- Graph Slice 风险可观测化：
  - 漏召回；
  - 重复节点；
  - 状态冲突；
  - 图漂移。
- 机械校验：
  - `entity_ref` 冲突检查；
  - `state` 冲突检查；
  - invalidates 指向检查；
  - orphan node 检查；
  - reconciliation warning；
  - graph lint report。

产物：

- Task Context Debug Report；
- Mermaid 图；
- Context Pack Trace；
- Reconciliation / Lint 报告。

---

## 第10讲：总结与路线图——完整任务态上下文系统闭环

**核心问题**：Task Graph 最终形成了一套怎样的 Agent 上下文工程方案？

主要内容：

- 回顾完整管线：

```text
用户对话
→ Graph Slice
→ Extractor
→ Patch
→ Applier
→ TaskGraph
→ Context Pack
→ 主 LLM
```

- 扩展闭环：

```text
TaskGraph → Memory Store → Recall → 新 TaskGraph
TaskGraph → Worker Context Pack → Worker Patch → Merge
Tool/File/Test Evidence → TaskGraph → Debug Report
```

- 最终 Demo：
  - 用户提出任务；
  - 约束逐步增加；
  - 旧方案被推翻；
  - 新方案形成；
  - 工具证据入图；
  - Context Pack 喂给主 LLM；
  - Memory 沉淀并在新会话召回；
  - Debug Report 定位上下文错误。
- 明确边界：
  - Task Graph 是任务态上下文管理机制；
  - Compact 是 token 压力下的执行层机制；
  - Memory 是跨会话沉淀层；
  - Debugger 调试的是工程中间产物；
  - Graph Slice 提升可运行性，但不承担最终正确性。

产物：

- 完整课程项目结构；
- 最终 Demo 流程；
- 工程路线图；
- 产品化方向；
- 研究问题清单。