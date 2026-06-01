以下是当前任务的结构化上下文，请基于它继续执行任务。

任务 ID：task_001
当前轮次：8
当前焦点节点：n_0049
当前焦点：确认/backtests/execution-config端点的返回形态（结构化BacktestResult JSON或markdown）以完成call_backtest.py设计

## 任务目标
- 让AOS真正服务于量化主线，通过Strategy Research Agent产出真实策略档案
- 复活Strategy Research Agent作为AOS的第一个领域employee

## 有效约束
- 避免基础设施惯性：不能持续在基建层打转而忽略量化主线
- 避免身份混淆：AOS是手段不是目的，量化研究是核心使命
- 避免半成品税：Strategy Research Agent搁置越久，上下文重建成本越高
- 一级风险：结构化输出稳定性、假设生成语义多样性、失败循环退出条件
- 二级风险：prompt资产管理、token成本失控、回测耗时

## 当前有效决策
- 采用Structured Output（Function Calling）强制约束策略IR输出，替代自由文本解析
- 采用已有策略向量空间统计+排斥规则prompt提升假设多样性
- 退出条件使用三道闸：迭代硬上限10次、token预算上限、连续3次假设未通过验收 → 更新为具体阈值：LLM调用≤20次/工单，回测≤10次/工单，连续5轮无改进（改进定义为+5%相对增幅）
- Prompt资产管理使用Git+物理多文件，每次修改新建文件不覆盖
- 增加Token监控装饰器，写入metrics文件并由duty_reporter日报汇报
- 阶段1只测量回测耗时，不优化
- 工单驱动流程：Boss建工单，手动触发Agent，Agent跑完后写回执，Boss review
- 入库标准直接引用data/knowledge/schema.md第5节（Lint清单及指标合法性等）
- data/knowledge目录保持原位不移动，作为领域知识层；整体采用四层物理布局：组织契约层(aos/)、智能体能力层(openclaw_skills/)、领域知识层(data/knowledge/)、架构文档层(docs/)
- strategy-researcher的scripts/应包括research_loop.py, hypothesis_gen.py, kb_query.py, 以及可选的kb_write.py
- strategy-researcher调用既有HTTP API（/strategy-builder/invoke, /backtests/execution-config），不独立实现策略构建和回测
- 入库标准直接引用data/knowledge/schema.md第5节（Lint清单：4位小数、不删除记录、Period一致性、净值穿零N/A标注等），不另设标准
- strategy-researcher脚本精简为research_loop.py（主循环）、hypothesis_gen.py（LLM假设生成）、kb_query.py（知识库只读查询），可选kb_write.py（若知识库函数非HTTP）

## 相关事实
- 当前真实处境：主线量化研究停滞，AOS框架搭好了但只有duty_reporter v0落地，Strategy Research Agent被设计但未真正跑起来
- 第一阶段积累的资产包括schema.md、回测API、4条标杆策略、研究方法论约束
- Claude推荐的路线图：阶段0（duty_reporter维稳）、阶段1（复活Strategy Research Agent）、阶段2（根据产出决定下一个employee）、阶段3（指挥舱MVP不早于2个月后）
- 阶段1的成功定义：1)产出≥10个入库策略档案；2)自主性：无需中途干预跑完≥5次假设循环；3)可信性：指标自洽，IR语义正确；4)AOS-native：通过AOS工单驱动，日报汇报；5)可复盘：每次迭代有完整trace
- OpenClaw框架下，SKILL.md必须放在openclaw_skills/<skill名>/目录下，与TOOLS.md和scripts/同级
- quant_assistant已有脚本：order_execute.py, strategy_builder.py, strategy_deploy.py，与strategy-researcher可能存在能力重叠
- schema.md是Agent操作契约，定义了研究循环、回测产物结构、知识库函数（create_strategy_archive, append_backtest_result, log_research_event, update_index等）
- quant_assistant与strategy-researcher为平行Agent，共享后端API，分别服务人驱（飞书）和自驱（ticket），不互相调用
- 用户确认HITL阈值：LLM调用≤20次/工单，回测≤10次/工单，连续5轮无改进（改进定义为当前工单内最优轮Sharpe相对增幅≥+5%）
- 用户确认试运行期为3个工单
- TKT-2026-002（Charter起草）已完成落盘，包括Charter、SKILL.md、TOOLS.md等文件已创建
- 当前阻塞链：TKT-2026-003（后端KB API）处于open状态，阻塞TKT-2026-004（脚本实现）
- knowledge_base.py位于quant_intelligence/strategy_builder/knowledge_base.py
- TKT-2026-003已实现：后端知识库API（/knowledge/index, /knowledge/archives, /knowledge/archives/{id}, /knowledge/log）已部署，返回markdown原文或档案列表
- 知识库API实际返回格式：read_index()和read_log()返回str（markdown原文），list_strategy_archives()返回list[{strategy_id, content, file_path}]，read_strategy_archive(id)返回{strategy_id, content, file_path}，content均为markdown文本
- Windows后端实际可达地址为 http://192.168.1.136:8000

## 待办事项
- 整理ROADMAP_v2.md写入aos/projects/abu_modern/
- 开顶层工单TKT-2026-002: Strategy Research Agent v1 - 阶段1启动，含子工单按周拆分，验收标准为阶段1五条成功定义
- 在aos/runtime/里创建_frozen_ideas.md，冻结Wiki Agent、指挥舱UI等诱惑
- 决定阶段1使用的LLM选型（供应商、API、预算约束、cron环境可用性）
- TKT-2026-004工单已起草：strategy-researcher脚本实现（call_builder.py, call_backtest.py, kb_query.py, smoke_http_clients.sh），后端地址默认http://192.168.1.136:8000，支持环境变量覆盖，统一输出格式和退出码约定，待项目AI实施
- 确认/backtests/execution-config端点的返回形态（结构化BacktestResult JSON或markdown）以完成call_backtest.py设计

## 相关文件
- ROADMAP_v2.md
- openclaw_skills/strategy-researcher/SKILL.md
- openclaw_skills/strategy-researcher/TOOLS.md
- aos/org/agents/agent-strategy-researcher.md (Charter v0.1.0)
- strategy-researcher/scripts/call_builder.py
- strategy-researcher/scripts/call_backtest.py
- strategy-researcher/scripts/kb_query.py
- strategy-researcher/scripts/smoke_http_clients.sh

## 已推翻或废弃内容
- 已推翻：AOS-native目录结构：org/agents/agent-strategy-researcher.md (Charter)，projects/abu_modern/aos/agents/strategy-researcher/ 下含 SKILL.md, prompts/, runs/, README.md
  - 原因：图中未记录明确 invalidates 原因，只知道该节点 status=superseded。

请注意：
- 不要继续采用已推翻或 superseded 的方案。
- 必须遵守有效约束。
- 优先围绕当前焦点继续推进。