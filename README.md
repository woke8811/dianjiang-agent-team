# Dianjiang · Multi-Agent Team Workflow for WorkBuddy

> 点将台 —— 人在环上的多智能体协同分工工作流 / Human-in-the-loop multi-agent collaboration skill with independent review, evidence-first handoffs and anti-loop guardrails.

首席于**点将台**派活，执行席出征制件，独立审核席复核，捷报交回中军帐。名字背后的四个动作，正是这套 skill 的完整闭环。

脱胎于「一人成队」MAS 方法学，融合 ruflo 的工程理念，由 WorkBuddy 重新组织并落地到原生能力之上。零外部依赖，即装即用。

**关键词 / Keywords**: multi-agent collaboration · AI agent team · human-in-the-loop · agent delegation · independent review · agentic workflow · WorkBuddy skill · prompt engineering

> 仓库名中的 `agent-team` 是为了让人搜 "multi-agent / AI agent team" 时能找到本项目；skill 内部名仍为 `dianjiang`（点将台），触发词不变。

## 为什么做这个

多智能体协作最大的问题从来不是"能不能分工"，而是分工之后：**责任不清、重复审核、反复退修越改越偏、授权被 AI 擅自扩大**。这套 skill 给"一个人带多个 AI 助手干活"补齐四件事——**责任边界、独立互检、证据链、授权可回查**。

## 先说实话：多智能体不省 token

每个子代理都有自己的上下文，分派、汇报、互检都是开销。多智能体通常**更费 token**，它的价值在质量、可审计、防串味，不在省钱。真正的省 token 来自上下文隔离和"该单干时别起队"。所以这套 skill 内置了成本纪律：简单任务直接单干。

## 核心闭环

**派 → 接 → 做 → 核 → 判 → 交**，每一步有记录、有证据、有明确责任人。

```
用户定目标
  ↓
首席确定实际执行席（按任务选，不总交生产席）
  ↓
执行席制件（开工前查重，只做受影响部分，缺事实报缺不编造）
  ↓
另一个真实上下文的独立审核席核验成品
  ├─ 通过 → 首席核收 → 原会话交回 → 有权者验收
  ├─ 退回修改 → 原岗只改受影响处（≤2次）→ 重新独立核验
  └─ 提交首席 → 按标准处置，必要时交本人拍板
```

## 安装

### WorkBuddy

克隆本仓库后，把仓库内的 skill 文件复制到技能目录并改名为 `dianjiang`（与 SKILL.md 的 `name` 一致）：

```bash
git clone https://github.com/woke8811/dianjiang-agent-team.git
# 用户级（所有项目可用）
cp -r dianjiang-agent-team/ ~/.workbuddy/skills/dianjiang/
# 或项目级（随项目共享）
cp -r dianjiang-agent-team/ <project>/.workbuddy/skills/dianjiang/
```

新开会话即可加载。说"用点将台分工""起三席"即可触发，简单任务它会自动判断为单干。

### GitHub Topics（维护者用）

发布后在仓库 Settings 里添加这些 topics，它们是 GitHub 站内发现的主要入口：

`multi-agent` `ai-agents` `agentic-workflow` `human-in-the-loop` `agent-orchestration` `prompt-engineering` `workbuddy` `workbuddy-skill` `ai-agent-framework` `llm` `collaboration` `multi-agent-systems`

### 其他平台

`SKILL.md` + `references/` 是标准 Skill 格式（name/description frontmatter），可作为提示词知识喂给兼容 Skill 的 Agent；`references/templates.md` 里的主提示词可整段复制到任意大模型使用。

## 目录结构

```
dianjiang/
├── SKILL.md                        主入口：定位、铁律、成本纪律、三席闭环、四道闸、拍板命令
└── references/
    ├── philosophy.md               六条设计哲学 + 与原版 MAS / ruflo 的借鉴对照与署名
    ├── workflow.md                 LOOP 闭环、6 状态机、证据先行、退修循环与收敛控制
    ├── roles.md                    16 席岗位卡（职责/交件/退回权/不管）
    ├── guardrails.md               权限授权边界、隐私、诚实边界、AI 判定清单
    ├── templates.md                主提示词、启动词、派单/回件/登记行模板
    └── workbuddy-native.md         WorkBuddy 原生落地指南与完整协作范例
```

## 防死循环机制（本项目重点）

多智能体最常见的翻车是"审→退→改→再审"无限循环、为小问题朝错误方向反复迭代。为此设计了五道闸：

1. **退修断路器**：同一任务最多退修 2 次；每次退回必须写清"具体位置 + 可验证判据 + 证据"三要素，模糊退回无效。第 3 次触发即强制熔断，停止自动循环转人工。
2. **收敛判据**：每轮送审前检查改动是否更接近锁定项；判定"原地打转 / 方向偏了 / 越改越多"就立即叫停。
3. **审核去重**：成果结论权归审核席独有，督查席只核流程不碰成果，杜绝两个席重复核成品、结论打架。
4. **开工查重**：执行席开工前先查共享记忆与台账，有同类已核成果则复用，不重复造轮子。
5. **`/拍板` 兜底命令**：本人随时说"拍板"，立即冻结所有自动循环，把争议点和 2–3 条可选路径交本人裁决；决定前不擅自推进。另补齐了原版空白的"提交首席"分支处置流程。

## WorkBuddy 原生映射

| MAS 抽象概念 | WorkBuddy 原生落地 |
|---|---|
| 首席统筹 | 当前主会话（主控 Agent） |
| 执行席 / 审核席 | `Agent` 工具启动的独立子代理（保证真实独立上下文） |
| 任务台账 | 任务系统 + 工作区持久台账文件 |
| 共享项目记忆 | 工作区 `.workbuddy/memory` 三层记忆 |
| 群 / 席名册 | 逻辑分组（非真实聊天室） |

## 设计来源与署名

- **组织方法学**借鉴 [大冰 / solo-to-team-skills](https://github.com/dabingfy-dot/solo-to-team-skills)（MIT）——保留"人在环上 / 独立核验 / 原岗退修 / 诚实边界"核心精神。
- **工程理念**借鉴 [ruvnet/ruflo](https://github.com/ruvnet/ruflo)（原 Claude-Flow，MIT）的公开文档——能力分层、共享记忆减少重复、最小可用起步、成本可观测、改进需对照基线。未引入其代码、MCP 服务或 npm 依赖。
- **WorkBuddy 原生版（点将台 / Dianjiang）**为重新组织与原创实现，其中防死循环四道闸、`/拍板` 兜底命令、WorkBuddy 原生映射为本版原创增强。

## 许可

[MIT](LICENSE) —— 使用时请保留上述来源署名。
