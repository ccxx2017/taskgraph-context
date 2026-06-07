# SYSTEM MESSAGE

你是 Task Graph Extractor。

```
你的任务不是回答用户、不解决问题、不给建议。
你只从本轮输入中抽取 TaskGraph 的增量补丁 Graph Patch，并只输出合法 JSON。

重要前提：
当前提供的不是完整 TaskGraph，而是本轮相关上下文 Slice。
你只能引用 Slice 中已出现的旧节点 ID，或本轮新建的节点 ID；
不得臆造未提供的旧节点 ID。全图级遗漏由后续 reconcile 检查。
```

## 一、节点类型 (type)

UserGoal | Constraint | Fact | ToolResult | Decision | FileArtifact | OpenTask

## 二、节点结构

new_nodes:
```json
{
  "node_id": "n_0001",
  "type": "<上述七种之一>",
  "content": "节点内容",
  "entity_ref": "稳定实体标识，无则 null",
  "state": "open|blocked|in_progress|implemented|deployed|resolved|cancelled|unknown|null",
  "status": "active"
}
```

updated_nodes:
```json
{ "node_id": "n_0001", "changes": { "content": "...", "entity_ref": "...", "state": "...", "status": "active|superseded|resolved" }, "reason": "为什么更新" }
```

superseded_nodes:
```json
{ "node_id": "n_0001", "reason": "为什么废弃" }
```

new_edges:
```json
{ "source": "n_0001", "target": "n_0002", "relation": "refines|depends_on|supports|invalidates|implements|serves|produces|derived_from" }
```

## 三、entity_ref 与 state

- entity_ref：用于机械判断两个节点是否同一长期实体。可填工单号 / API 路径 / 文件路径 / 脚本名 / 课程编号 / 模块名 / 节点编号。可长期追踪的节点必须尽量填写；无法确定填 null；禁止把普通描述性文本当作 entity_ref。
- state 归一化：未完成/待实现/todo/pending→open；阻塞/卡住/依赖未完成→blocked；进行中→in_progress；已实现/代码已完成→implemented；已部署/已上线/已落盘→deployed；已完成/已验证/closed/done→resolved；取消/不做了/废弃→cancelled；有状态含义但判断不了→unknown；无状态含义→null。

## 四、Patch 总结构（顶层字段不可省略）

```json
{ "task_id": "task_xxx", "turn_id": 1, "new_nodes": [], "updated_nodes": [], "superseded_nodes": [], "new_edges": [] }
```

## 五、ID 规则

新节点从系统给的 next_node_id 起递增；不复用、不臆造旧 ID；superseded_nodes 与边的 source/target 只能引用 Slice 中已有节点或本轮新节点。

## 六、生命周期判定（先对齐 Slice 中所有 active 节点，尤其 conflict_candidates，再决定动作）

对每个候选信息只能选一种动作：

1. SAME_ENTITY_MORE_DETAIL：同一实体仅更具体/更准确 → updated_nodes，不新建。
2. REPLACES_OLD_ENTITY：使某 active 节点不再成立、被替代、状态互斥迁移或被否定 → 旧节点入 superseded_nodes + 新建 new_node + 加 invalidates 边。
3. DEPENDS_OR_SUPPORTS_EXISTING：是旧节点的原因/依据/约束/实现/结果/子任务/支撑 → 新建 new_node 并用边连接。
4. NEW_ENTITY：无法与任何 active 节点建立上述关系时才新建；除根 UserGoal 外，新节点必须至少有一条边连入图。

## 七、事实一致性反向检查（关键）

按优先级判同一实体：entity_ref 精确匹配 > node_id > 文件/API/工单号词面 > 最近节点 > 关键词。

若 new_node.entity_ref == old_node.entity_ref 且 state 互斥，即视为冲突；即使用户未明说“推翻/废弃”，也必须 superseded 旧节点并加 invalidates 边。

必须捕获的互斥迁移（旧→新）：
- open/blocked/in_progress/pending/todo → implemented/deployed/resolved（工单完成）
- “A 阻塞 B / B 依赖未完成的 A” → “A 已实现/部署/完成”（阻塞失效）
- “接口返回格式待确认” → “返回格式已明确”（OpenTask 转 resolved 或 superseded）
- “采用方案 A” → “改用 B / A 不可行”（方案替代）
- “文件/目录位于 A” → “改为 B”且二者互斥（位置替代）

invalidates 边格式：
```json
{ "source": "新节点ID", "target": "旧节点ID", "relation": "invalidates" }
```
reason 须写明：同一实体状态从 X 变为 Y / 新事实使旧阻塞关系不再成立 / 新方案替代旧方案 / 新路径替代旧路径。

## 八、硬性禁止

不因措辞变化重复建节点；同一任务/决策/约束/entity_ref 不得有多个互斥 active 版本；除根 UserGoal 外不得有孤立节点；不引用 Slice 外的旧 ID；只输出 JSON，无任何解释文字。

请只输出 Graph Patch JSON。

# USER MESSAGE

你现在收到本轮 Task Graph Extractor 的动态输入。

请严格遵守 system message 中的规则。
你不是回答用户，而是只输出 Graph Patch JSON。

## Runtime Control

task_id: task_0001
turn_id: 6
next_node_id: n_0034

## extractor_context_pack.json

```json
{
  "task_id": "task_0001",
  "turn_id": 6,
  "root_goals": [
    {
      "node_id": "n_0001",
      "type": "UserGoal",
      "content": "从策略发现到数字组织演进，实现量化研究的知识积累循环",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "standing_constraints": [
    {
      "node_id": "n_0020",
      "type": "Constraint",
      "content": "入库标准：IR编译无静默警告；训练集和测试集均≥5笔已平仓交易；核心指标数值合法；样本内外差异合理；分析章节由LLM自动生成并标注[auto-analysis]",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "active_open_tasks": [
    {
      "node_id": "n_0026",
      "type": "OpenTask",
      "content": "建立AOS目录骨架：创建org/agents/agent-strategy-researcher.md和openclaw_skills/strategy-researcher/完整结构（含SKILL.md、TOOLS.md、scripts/），而非之前建议的projects/abu_modern/aos/agents/strategy-researcher/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0015",
      "type": "OpenTask",
      "content": "开一张工单：TKT-2026-002 'Strategy Research Agent v1 - 阶段 1 启动'，包含顶层工单及按周拆分子工单，验收标准为阶段1五条成功定义。",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0031",
      "type": "OpenTask",
      "content": "第4周AOS闭环交付：duty_reporter完整集成研究进度，写阶段1 retrospective",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0030",
      "type": "OpenTask",
      "content": "第3周稳定性交付：连续5张不同意图工单，每张≥1入库策略，无人工干预",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0029",
      "type": "OpenTask",
      "content": "第2周Loop交付：完整的5+假设自主循环工单，至少1个入库策略",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0028",
      "type": "OpenTask",
      "content": "第1周地基交付：Structured Output demo、token监控、AOS目录骨架、单次闭环脚本",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0027",
      "type": "OpenTask",
      "content": "实现工单驱动循环：解析工单意图与约束，循环生成假设、回测、分析、判断，产出归档到runs/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0025",
      "type": "OpenTask",
      "content": "实现Token监控装饰器：记录每次LLM调用的token消耗，写入runtime/metrics/llm_usage/，并集成到duty_reporter日报",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0024",
      "type": "OpenTask",
      "content": "建立Prompt资产管理：在prompts/下按版本号新建文件，维护CHANGELOG，使用软链接指向current",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0023",
      "type": "OpenTask",
      "content": "实现退出条件三道闸：硬上限迭代≥10、资源闸累计token超预算、价值闸连续3次假设未通过验收",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0022",
      "type": "OpenTask",
      "content": "实现假设多样性机制：注入已有策略统计摘要和排斥规则，避免重复生成相似策略",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0021",
      "type": "OpenTask",
      "content": "实现Structured Output改造：用Function Calling/response_format强制约束strategy_ir输出，替代自由文本解析",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "recent_nodes": [
    {
      "node_id": "n_0033",
      "type": "Fact",
      "content": "data/knowledge目录已存在并包含策略档案：strategies/下有5个stg_*.md文件，以及market_insights/、index.md、log.md、piercing_report.json、piercing_report.md、schema.md、semantic_phrase_coverage.md",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0032",
      "type": "Fact",
      "content": "OpenClaw框架下skill的实际位置：根目录D:\\智能投顾\\量化相关\\abu_modern\\openclaw_skills，包含duty-reporter、quant_assistant等skill，每个skill有SKILL.md、TOOLS.md和scripts/目录",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0031",
      "type": "OpenTask",
      "content": "第4周AOS闭环交付：duty_reporter完整集成研究进度，写阶段1 retrospective",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0030",
      "type": "OpenTask",
      "content": "第3周稳定性交付：连续5张不同意图工单，每张≥1入库策略，无人工干预",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0029",
      "type": "OpenTask",
      "content": "第2周Loop交付：完整的5+假设自主循环工单，至少1个入库策略",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0028",
      "type": "OpenTask",
      "content": "第1周地基交付：Structured Output demo、token监控、AOS目录骨架、单次闭环脚本",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0027",
      "type": "OpenTask",
      "content": "实现工单驱动循环：解析工单意图与约束，循环生成假设、回测、分析、判断，产出归档到runs/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0026",
      "type": "OpenTask",
      "content": "建立AOS目录骨架：创建org/agents/agent-strategy-researcher.md和openclaw_skills/strategy-researcher/完整结构（含SKILL.md、TOOLS.md、scripts/），而非之前建议的projects/abu_modern/aos/agents/strategy-researcher/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0025",
      "type": "OpenTask",
      "content": "实现Token监控装饰器：记录每次LLM调用的token消耗，写入runtime/metrics/llm_usage/，并集成到duty_reporter日报",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0024",
      "type": "OpenTask",
      "content": "建立Prompt资产管理：在prompts/下按版本号新建文件，维护CHANGELOG，使用软链接指向current",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0023",
      "type": "OpenTask",
      "content": "实现退出条件三道闸：硬上限迭代≥10、资源闸累计token超预算、价值闸连续3次假设未通过验收",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0022",
      "type": "OpenTask",
      "content": "实现假设多样性机制：注入已有策略统计摘要和排斥规则，避免重复生成相似策略",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0021",
      "type": "OpenTask",
      "content": "实现Structured Output改造：用Function Calling/response_format强制约束strategy_ir输出，替代自由文本解析",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0020",
      "type": "Constraint",
      "content": "入库标准：IR编译无静默警告；训练集和测试集均≥5笔已平仓交易；核心指标数值合法；样本内外差异合理；分析章节由LLM自动生成并标注[auto-analysis]",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0019",
      "type": "Fact",
      "content": "二级风险：prompt资产版本管理、token成本失控、回测耗时",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0018",
      "type": "Fact",
      "content": "一级风险：结构化输出稳定性、假设生成语义多样性、失败循环退出条件",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "symbol_hits": [
    {
      "node_id": "n_0026",
      "type": "OpenTask",
      "content": "建立AOS目录骨架：创建org/agents/agent-strategy-researcher.md和openclaw_skills/strategy-researcher/完整结构（含SKILL.md、TOOLS.md、scripts/），而非之前建议的projects/abu_modern/aos/agents/strategy-researcher/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0032",
      "type": "Fact",
      "content": "OpenClaw框架下skill的实际位置：根目录D:\\智能投顾\\量化相关\\abu_modern\\openclaw_skills，包含duty-reporter、quant_assistant等skill，每个skill有SKILL.md、TOOLS.md和scripts/目录",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0033",
      "type": "Fact",
      "content": "data/knowledge目录已存在并包含策略档案：strategies/下有5个stg_*.md文件，以及market_insights/、index.md、log.md、piercing_report.json、piercing_report.md、schema.md、semantic_phrase_coverage.md",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0017",
      "type": "Fact",
      "content": "阶段1成功定义五条：1)产出≥10个入库策略档案；2)自主性：Boss提工单后无需干预跑完≥5次假设迭代；3)可信性：抽查3个新档案指标自洽且IR语义正确；4)AOS-native：工单驱动，产出沉淀runtime，日报汇报进度；5)可复盘：每次迭代有完整trace",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0015",
      "type": "OpenTask",
      "content": "开一张工单：TKT-2026-002 'Strategy Research Agent v1 - 阶段 1 启动'，包含顶层工单及按周拆分子工单，验收标准为阶段1五条成功定义。",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "conflict_candidates": [
    {
      "node_id": "n_0026",
      "type": "OpenTask",
      "content": "建立AOS目录骨架：创建org/agents/agent-strategy-researcher.md和openclaw_skills/strategy-researcher/完整结构（含SKILL.md、TOOLS.md、scripts/），而非之前建议的projects/abu_modern/aos/agents/strategy-researcher/",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0032",
      "type": "Fact",
      "content": "OpenClaw框架下skill的实际位置：根目录D:\\智能投顾\\量化相关\\abu_modern\\openclaw_skills，包含duty-reporter、quant_assistant等skill，每个skill有SKILL.md、TOOLS.md和scripts/目录",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0033",
      "type": "Fact",
      "content": "data/knowledge目录已存在并包含策略档案：strategies/下有5个stg_*.md文件，以及market_insights/、index.md、log.md、piercing_report.json、piercing_report.md、schema.md、semantic_phrase_coverage.md",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0017",
      "type": "Fact",
      "content": "阶段1成功定义五条：1)产出≥10个入库策略档案；2)自主性：Boss提工单后无需干预跑完≥5次假设迭代；3)可信性：抽查3个新档案指标自洽且IR语义正确；4)AOS-native：工单驱动，产出沉淀runtime，日报汇报进度；5)可复盘：每次迭代有完整trace",
      "entity_ref": null,
      "state": null,
      "status": "active"
    },
    {
      "node_id": "n_0015",
      "type": "OpenTask",
      "content": "开一张工单：TKT-2026-002 'Strategy Research Agent v1 - 阶段 1 启动'，包含顶层工单及按周拆分子工单，验收标准为阶段1五条成功定义。",
      "entity_ref": null,
      "state": null,
      "status": "active"
    }
  ],
  "recent_edges": [
    {
      "source": "n_0033",
      "target": "n_0001",
      "relation": "supports"
    },
    {
      "source": "n_0032",
      "target": "n_0026",
      "relation": "refines"
    }
  ]
}
```

## turn_text

````markdown
# 用户（同意B方案）：
好的，同意1. B方案；2. 确定HITL的建议阈值；3. 可以将运行期定为"3 个工单"
# Claude Opus4.7:
## 三项确认收到，锁定方案 → 准备落盘

三条确认我都记录了：

1. **架构：B 方案**（后端自动落库 + Agent 只读 4 个 KB API）
2. **HITL 阈值**：LLM 20 次 / 回测 10 次 / 连续 5 轮无改进（改进定义为"核心指标相对上一最优轮 < 5%"）
3. **试运行期：3 个工单**

下面我做四件事：**(a)** 把上一版的三份草案做一次 **diff 级清理**（几处我自己回看发现的小瑕疵）；**(b)** 明确**落盘顺序**和**每份文件的去向**；**(c)** 起草 **TKT-2026-003 的意图稿**（不完整工单，留给你确认后再正式建档）；**(d)** 列出 **Boss 你现在需要做的 5 个动作**。

---

## 一、上一版草案的自查清理

我回看自己写的三份文档，发现 3 处需要修正，先列出来免得你按原样落盘。

### 修正 1 · Charter §4.4 阈值表述

原稿只说"累计 LLM 调用 > 20"，没定义什么算"一次 LLM 调用"，也没定义"改进 < 5%"的基准指标。按确认补齐：

> ### 4.4 Human-in-the-loop 清单（**修正版**）
> 本员工**无需 Boss 批准即可执行**的动作已在 4.1~4.3 列尽。
> 以下三条为**硬阈值**，触达即暂停工单、等 Boss 指示：
>
> - **LLM 调用 ≤ 20 次 / 工单**。计数口径：`POST /strategy-builder/invoke` 的次数。builder 内部若触发多次 LLM 推理，计作 1 次。
> - **回测 ≤ 10 次 / 工单**。计数口径：`POST /backtests/execution-config` 的 200 响应数。超时或 5xx 不计。
> - **连续 5 轮无改进**。改进定义：当轮回测的 `sharpe_ratio`（或工单 Spec 中显式指定的目标指标）相比**当前工单内最优轮** ≥ +5% 相对增幅，才算改进。暂停时 worklog 写明"best_round=N, last_5_rounds_sharpe=[...]"。
>
> 触达任一阈值 → `research_loop.py` 立即 break，worklog 追加 `paused_for_boss_review`，git commit 推送，等待 Boss 改工单状态或人工指示。

### 修正 2 · Charter §6.1 心跳信号表述模糊

原稿写"接受工单后 2 小时内必须开始第一轮"——但 researcher 是 ticket-driven、一次启动一次性跑完的，没有"接受工单"和"开始执行"的时差概念。改为：

> ### 6.1 心跳信号（**修正版**）
> 本员工是 ticket-driven 单次执行模型（OpenClaw 调度 → 跑完 → 退出），不适用持续心跳。代之以**单工单内可观测的进度信号**：
>
> - 每轮结束必须 `git commit + push` 一次，Boss 可通过 aos 仓库 commit 历史追踪进度
> - 每轮结束必须 append 一条 worklog 到原工单（格式：`### {ts} @agent-strategy-researcher · round {N}`）
> - 单工单总 wall-clock **上限 4 小时**，超时由 OpenClaw 侧 timeout 强制终止，worklog 标记 `wall_clock_exceeded`

### 修正 3 · SKILL.md 主流程伪代码的"改进检查"位置错了

我原稿把阈值检查放在 `4.6` 每轮末尾，但"LLM ≤ 20 / 回测 ≤ 10"是**跨轮累计**判断，放在每轮末尾没问题；"连续 5 轮无改进"需要至少有 5 轮数据，应当从 round_6 开始才做检查。补一个 `check_stop_conditions` 的准确伪代码：

```python
# check_stop_conditions(rounds_so_far) 伪代码
def check_stop_conditions(rounds):
    llm_calls  = sum(r.llm_call_count for r in rounds)
    bt_calls   = sum(1 for r in rounds if r.backtest_ok)
    if llm_calls >= 20:  return "llm_quota_exceeded"
    if bt_calls  >= 10:  return "backtest_quota_exceeded"
    if len(rounds) >= 6:
        best_sharpe_before_last5 = max(r.sharpe for r in rounds[:-5] if r.sharpe is not None)
        last5_best = max((r.sharpe for r in rounds[-5:] if r.sharpe is not None), default=None)
        if last5_best is None or last5_best < best_sharpe_before_last5 * 1.05:
            return "no_improvement_5_rounds"
    return None
```

这段**不用塞进 SKILL.md 正文**，只是给 TKT-2026-004 实施时留个精确约束。我会把它作为**验收标准**写进 TKT-2026-004 的 Deliverable Spec。

---

## 二、落盘顺序与文件去向

按 aos 规约，三份文件各有位置。**请按这个顺序落**（顺序不能乱，因为 Charter 要先存在，SKILL.md 才能在 front matter 里 `charter_ref` 指到它）：

| # | 文件 | 绝对路径 | 动作 |
|---|------|---------|------|
| 1 | 工单本体 | `/home/ccxx/aos_repo/aos/runtime/tickets/open/TKT-2026-002-charter-strategy-researcher.md` | 新建 |
| 2 | Charter | `/home/ccxx/aos_repo/aos/org/agents/agent-strategy-researcher.md` | 新建（应用上面修正 1 & 2）|
| 3 | Agent 索引登记 | `/home/ccxx/aos_repo/aos/org/agents/_index.md` | **追加一行** |
| 4 | SKILL.md | `/home/ccxx/.openclaw/workspace/skills/strategy-researcher/SKILL.md` | 新建（应用上面修正 3 的 check_stop_conditions 思路，但正文伪代码保持骨架级）|
| 5 | TOOLS.md | `/home/ccxx/.openclaw/workspace/skills/strategy-researcher/TOOLS.md` | 新建（占位版） |
| 6 | scripts 占位 | `/home/ccxx/.openclaw/workspace/skills/strategy-researcher/scripts/.gitkeep` | 新建空文件 |
| 7 | 工单 worklog | 回到第 1 步的工单文件 | append "approved by @boss" 那一条 |
| 8 | git commit | aos 仓 + openclaw skills 仓（如分仓）| 分别 commit |

**要不要我把三份修正后的完整文档再贴一次？** 如果要，我直接贴 Charter 修正版 + SKILL.md 修正版（TOOLS.md 和工单上一版已经可以直接用），避免你手工合并 diff。

---

## 三、TKT-2026-003 的意图稿（只写 Intent + Spec 骨架，先不建档）

这张是**后端工单**，不归 strategy-researcher 管、也不归 duty-reporter 管，但它是 TKT-2026-002 → TKT-2026-004 之间的**硬前置**。先把它的轮廓拎清楚，避免你后续自己起草时又要重新想一遍：

> **TKT-2026-003 · 后端：知识库只读 HTTP API + 写侧自动落库**
>
> **Intent**：
> "strategy-researcher 要能读 `data/knowledge/` 里的策略档案和索引，但不能直接 import Python 模块。给后端加 4 个只读 GET；同时把 `create_strategy_archive` / `append_backtest_result` 内嵌到 `strategy-builder/invoke` 和 `backtests/execution-config` 的完成路径里，让知识库随业务流自动增长。"
>
> **Deliverable Spec（草案）**：
> - [ ] 新增 `GET /api/v1/knowledge/index` → 返回 `read_index()` 的纯文本
> - [ ] 新增 `GET /api/v1/knowledge/strategies` → 返回 `list_strategy_archives()` 的 JSON（每项含 id / intent / ir_summary / backtest_count）
> - [ ] 新增 `GET /api/v1/knowledge/strategies/{strategy_id}` → 返回 `read_strategy_archive()` 的 JSON（完整 markdown 内容）
> - [ ] 新增 `GET /api/v1/knowledge/log` → 返回 `read_log()` 的纯文本
> - [ ] 在 `strategy-builder/invoke` 成功生成新策略 IR 的路径上，调用 `create_strategy_archive()`
> - [ ] 在 `backtests/execution-config` 返回 200 的路径上，调用 `append_backtest_result()`
> - [ ] 在以上两个写侧点完成后，调用 `update_index()` 刷新 `index.md`
> - [ ] 所有 4 个 GET 接口在 `data/knowledge/` 不存在时返回 200 + 空/占位内容（而非 500）
>
> **显式排除**：
> - ❌ 不开放 POST / PUT / DELETE（写侧永不对 Agent 开放）
> - ❌ 不修改 `knowledge_base.py` 本身的 9 个函数签名
> - ❌ 不引入鉴权（内网 192.168.1.136 同网段信任模型，与其他 API 一致）

这张工单的执行人是**你（Boss）** 或未来的"后端维护 Agent"，不是 strategy-researcher。要不要我按 `_template.md` 给出完整工单文件？

---

## 四、Boss 你现在要做的 5 个动作

为了让 TKT-2026-002 真正进入 `in-progress`，需要你先做这 5 件事。每件都不超过 2 分钟：

1. **确认上面三处修正**（修正 1~3）——有不同意的直接说，一处没说就按我的版本走。
2. **确认是否需要我重贴 Charter 修正版和 SKILL.md 修正版完整文件**（还是你自己 diff 合并）。
3. **确认 TKT-2026-003 的 Spec 草案**——尤其第 5、6 条的"自动落库时机"是否就设在你说的那两个端点、有没有漏的写入触发点。
4. **告诉我 openclaw skills 是否与 aos 同仓** —— 影响"两次 commit"还是"一次 commit"，以及落盘顺序 3 和 4 是否需要跨仓 push。
5. **给我 TKT-2026-002 的 assigned_to 最终值** —— 我草案里写的是 `boss`（因为 Charter 本身要人起草），但如果你想让一个叙事对称的"未来的 agent-org-maintainer"占位，也可以。默认 `boss`。

五条里 **1、4、5 是必答**，**2、3 是选答**（你可以说"先搁置 TKT-2026-003，等 002 落盘再谈"）。

等你回复后，我就按最终版一次性把 Charter + SKILL.md + TOOLS.md + 工单文件四份内容全部贴出来，你直接复制落盘即可。

````

请只输出 Graph Patch JSON。

