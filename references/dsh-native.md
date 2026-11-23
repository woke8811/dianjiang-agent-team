# DSH 宿主映射（dsh-native.md）

本 skill 原版跑在 WorkBuddy 宿主（`workbuddy-native.md`）。本文件把同一套 MAS 概念映射到 **DSH 原生工具**，说明在 DSH 上"具体调什么"。只映射 DSH 会话工具上下文中真实存在的工具；拿不准的写进「缺口」，不编造。

> 适用宿主：DSH（dsh-go 手机版 Ubuntu proot 环境，或 Web GUI）；skill 装于 `$DSH_HOME/skills/dianjiang/`，默认工作区 `/root/workspace`。
> 工具名注：本 skill 其他 reference 里出现的 `Agent` / `TaskCreate` 等 WorkBuddy 工具名，一律按本文件读作"语义映射"，DSH 里不存在这些工具名，不得照字面调用。

## 1. 能力分层对照（DSH 版）

| 层 | 是什么 | 在 DSH 里 |
|---|---|---|
| 模型 | 提供推理与语言能力 | DSH 会话正在运行的模型（provider/model 为会话级配置，skill 不改变；派子代理时首席可显式指定 provider/model） |
| 技能 | 教 Agent 怎么做事的剧本 | 本 SKILL.md + references；在 DSH 技能目录中登记，由 `skill` 工具按需加载 |
| 原生工具 | Agent 干活的能力 | `subagent` / `send_message` / `list_agents` / `interrupt_agent`；`todo_write`；`create_goal` / `update_goal`；`ctx_memory` 与 `mnemon_*`；`read` / `write` / `edit` / `glob` / `grep` / `bash` / `read_image`；`ask_user_question`；`present`；`schedule_*`；`workflow` |
| 运行时 | 真正执行的环境 | DSH 会话（dsh-go 手机 App 或 Web GUI）；一个会话 = 一个执行上下文，子代理各有自己的上下文 |

结论同原版：装 skill 只是拿到剧本；能否分工取决于 `subagent` 提供"真实独立执行上下文"——这是 DSH 里对应 WorkBuddy `Agent` 工具的那一层。

## 2. MAS 概念 → DSH 工具映射表

| MAS 概念 | DSH 原生落地 | 用途与注意 |
|---|---|---|
| 首席统筹 | 主会话（当前入口会话）自身 | 定目标、派活、核收、原会话交回；不亲自制成品、不自审；跨回合长任务可 `create_goal` 持久目标、卡住时 `update_goal`(blocked) 上报 |
| 执行席 | 一次**全新** `subagent` 调用 | 提示词必须自包含（角色卡 + 派单字段，见 §3）；子代理用 `send_message` 把回件报回首席 |
| 审核席 | **另一次全新** `subagent` 调用 | 只给成品路径 + 验收条件 + 原始目标，不带生产席自述（盲审）；`subagent_fork` 会继承主会话全部已完成回合，只用于"同席续做/退修"，**不用于盲审** |
| 原岗退修 | 对原生产子代理 `send_message(agent_id)` | 运行中孩子在下一步收到；已停止的孩子起新回合；只改受影响处，新版写回同一收件位置 |
| 叫停 / 熔断 | `interrupt_agent(agent_id)` | 停止运行中的子代理即冻结自动循环；被停的外部执行不能再收后续消息——先停，再决定重派 |
| 席名册 | `list_agents`（scope: children / descendants） | 活的 id / 标签 / 状态清单，仅本会话有效；跨会话名册靠台账文件的名册段 |
| 任务台账 | `todo_write`（会话内状态）+ 工作区 `.mas-ledger.md`（read/write/edit，只追加） | `todo_write` 不跨会话，状态机行统一记文件；过程事件（叫开工/续期/升级/点名）不改阶段 |
| 共享项目记忆 | `ctx_memory`（write / search / get / update / archive，`project` 参数可读写他项目记忆） | 本项目范围、与其他编码 Agent 共享的持久记忆；存"已核结论 + 版本 + 出处"；开工查重先 search 复用 |
| 项目级重要记录 / 持久洞察 | `mnemon_document_search` + `mnemon_document_create`；`mnemon_recall` / `mnemon_remember` | Documents 装可复用项目知识（先查后建，防重）；Memory Space 装跨会话持久证据 |
| /拍板 兜底 | `ask_user_question`（questions[].options=2–3 条路径；multi_select 可省） | 自写决定可走 dsh-ui `options` 的 freetext（dsh-ui 组件字段，非 `ask_user_question` 参数），或直接把拍板清单交回原会话由本人回复裁决；用户跳过（skip）不等于已决定，保持冻结 |
| 证据获取 | `read` / `glob` / `grep` / `bash` / `read_image`；后台结果 `job_output`；`ctx_expand` / `save_token_expand` 取回被丢弃的工具输出 | 审核席亲手打开同版成品；执行人报告只当线索；"实际读到的本版本"为准 |
| 原会话交回 | 主会话回复 + `present`（展示文件） | 用户点头才算真正关闭；验收权归有权者 |
| 大规模并行（扩席时） | `workflow` 脚本（`agent()` hook 扇出多个子代理） | 仅当用户明确要 workflow 或大规模多智能体编排时使用；三席闭环用不上 |
| 定时查点 | `schedule_create` / `schedule_list`（宿主会话具备时） | 到点向会话投递"查点"提示；宕机后可能重发、依赖宿主配置；不是"后台常驻团队" |

## 3. 三席最小闭环在 DSH 的一次完整跑法

前提：任务可分工且需互检（例："M-01 竞品调研并出结论页"）。交件一律"文件 + 消息"：派单 / 回件字段按 `templates.md` §4，登记行按 §5，统一记入 `.mas-ledger.md`（只追加、不改写）。

**第 0 步｜首席判定（主会话）**
先问"分工收益 > 协调开销吗？"只是查个价、改一行——单干，结束。可分工再继续。

**第 1 步｜登记**
- `todo_write`：建会话内清单（M-01 待收 / 待核 / 待验收）
- `write`/`edit`：向工作区 `.mas-ledger.md` 追加 `待收｜M-01｜…` 行（字段按 templates.md §5）
- 跨回合长任务：`create_goal`（objective；可选 max_goal_rounds）

**第 2 步｜派生产席**
- `subagent`（description "M-01 制件"；prompt = templates.md §3 角色模板**内联进提示词**：目标与锁定项 / 输入版本 / 应交物 / 收件位置 `/root/workspace/deliverables/M-01/v1.md` / 验收条件 / 授权出处 + "缺事实报缺、不编造；只做受影响部分"）
- 等完成通知（不轮询）；子代理把成品写到收件位置，经 `send_message` 回件（任务标识 / 实际执行者 / 成果版本 + 可访问入口 / 已做核验及证据 / 已知问题 / 当前阶段）
- 首席接件：`todo_write` → 已收到成果（待核）；台账追加 `已收到成果｜M-01｜路径｜待核`

**第 3 步｜独立审核（关键）**
- `subagent` 另起**全新**审核子代理（description "M-01 审核"），只给：成品路径 + 验收条件 + 原始目标；**不带**生产席自述；若它看过既有结论须声明"非盲审"
- 审核子代理 `read` 同版成品，核事实/标准/质量，经 `send_message` 回首席三选一："通过 / 退回修改 / 提交首席"+ 原因 + 位置 + 所核版本
- 首席接件：通过 → 台账追加 `已收到成果｜M-01｜路径｜核验通过，待有权者决定`；退回 → 追加 `退回｜M-01｜位置 + 判据 + 证据｜计数 n/2`

**第 4 步｜判与交**
- 通过：首席亲自 `read` 成品（证据先行五步，全篇看完）→ `todo_write` → 已验收（待有权者决定）→ 原会话交回（`present` 展示或给路径，请用户验收；dsh-go 手机版环境下给用户的最终成品按宿主约定放 `/sdcard/Download/DSH-Go/`，见 AGENTS.md）。用户点头才真正关闭
- 退回：`send_message(agent_id=原生产子代理)` 发退修单（具体位置 + 可验证判据 + 证据，缺项视为无效退回）；子代理只改受影响处、写新版 v2 回同一收件位置、回新回件 → 回到第 3 步，用**新**审核子代理再核
- 退修计数达 3（断路器触发）：立即停自动循环；运行中的孩子用 `interrupt_agent` 停；台账追加 `熔断｜M-01｜计数 n=3，停自动循环` → 转拍板

**第 5 步｜收尾与沉淀**
- 台账追加 `已验收｜M-01｜怎么核的｜接受者及决定/授权依据`；写收尾三模板（templates.md §4）
- `ctx_memory`（action: write）把"已核结论 + 版本 + 出处"写进项目记忆
- 可复用的重要项目知识：先 `mnemon_document_search` 查重，再 `mnemon_document_create`

**拍板（任何时刻可触发）**
- 本人说"拍板/交我决定"或熔断触发：立即**冻结**当前任务所有自动循环（停改、停审、停派）
- 整理拍板清单：①争议点 ②已尝试方案与结果 ③可选 2–3 条路径及各自代价
- `ask_user_question`（questions[].options=路径；multi_select 可省）；自写决定可走 dsh-ui `options` 的 freetext（dsh-ui 组件字段，非 `ask_user_question` 参数），或直接把拍板清单交回原会话由本人回复裁决；本人未给决定前不推进；**skip ≠ 决定**，保持冻结

## 4. 缺口与注意事项（WorkBuddy 版假设在 DSH 不成立之处）

1. **工具名不沿用**：`Agent` / `TaskCreate` / `TaskUpdate` / `TaskList` / `TaskOutput` 在 DSH 不存在；语义映射为 `subagent` / `send_message`（结果经 `send_message` 或完成通知取回）/ `todo_write` / `job_output`（后台任务结果）。
2. **`.workbuddy/memory/` 目录不存在**：不要创建它。DSH 的共享项目记忆走 `ctx_memory` 存储（项目级、与本机其他编码 Agent 共享）；热记忆是 `mnemon_runtime_memory` 托管的 USER.md / MEMORY.md；宿主文件位置互不混用。
3. **"全员自动加载"不成立**：子代理各有独立上下文，不会自动继承主会话已加载的 SKILL / 角色卡内容；首席必须把所需角色卡与模板文本**内联**进每个子代理的提示词。
4. **委派深度有上限**：DSH 按会话配置子代理嵌套深度上限（运行时上下文公布 current / maximum / remaining；如本环境 max 1：主会话可派直接孩子，孩子不能再派席）。三席闭环（首席=主会话、席=直接子代理）不受影响；16 席深层级、"执行席再开子席"可能超限——先查运行时上下文再假设，超限时改用 `workflow` 扇出或首席串行派席。
5. **`subagent_fork` 不能当审核席**：它给子代理注入全部已完成回合，盲审会被生产席自述污染；它只适合"同席续做 / 退修后续"。
6. **席名册不持久**：`list_agents` 仅本会话有效；跨会话名册靠 `.mas-ledger.md` 的名册段与登记行。
7. **危险动作闸门更紧**：部分 DSH 会话禁用审批提示，需审批的动作会被**自动拒绝**；子代理权限范围在派生时固定、不能在内部扩权。因此危险动作（删除 / 安装 / 对外发送 / 发布 / 凭据）不写进子代理提示词；子代理遇到即上报，首席按六条铁律统一"请本人来"。
8. **定时叫醒"部分可用"，不过度声称**：`schedule_*` 在宿主具备时可做定时查点提醒，但宕机后投递可能重发、依赖宿主配置；它不是"后台常驻团队"、不是私密通道，不据此声称未验证能力。
9. **宿主插件加的工具不做映射基座**：会话专属插件工具（如 douyin_*、mcp__phone__*）不在本表内；缺能力时用 `find_dsh_plugin` 找插件，但安装/卸载属危险动作 → 请本人来。
10. **被丢弃的证据可取回，但以"实际读到"为准**：DSH 上下文管理会把旧工具输出压缩/丢弃（`[dropped §N§]`、save-token 提示）；重核证据时先用 `ctx_expand` / `save_token_expand` 取回原文，再对照当前版本成品核验，不凭记忆中的旧输出下结论。

11. **子代理会"无声失败"（不稳定渠道下高发）**：子代理是多轮会话，渠道中途 5xx / 429 / 超时会掐掉整席，完成回执报 `failed` 且可能丢失；但**部分工作可能已落盘、结论可能迟到才送达**（本试验实测 4 席 3 次"失败"回执，实际活全部干完）。恢复规程：① 不轻信回执，先证据先行自查（亲手打开成品、grep / diff 核实际状态）再定下一步；② 需续做就重派**全新**子代理（原席已死，`send_message` 只续活席）；③ 同一类无声失败反复出现 → 把子代理改路由到更稳的渠道（`subagent` 调用可传 provider/model 覆盖），或小额收尾工作由首席直接完成并在交回时声明"首席代做"偏差；④ 连续 3 席失败即冻结自动循环，按 /拍板 交本人定线路。

> 兜底原则：以上任何映射若与当前宿主不符，**缺事实就报缺**——以该会话实际工具列表为准，缺的能力明写"未落地"，不虚构通道。
