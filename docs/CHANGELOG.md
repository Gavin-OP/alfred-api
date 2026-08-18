---
type: reference
status: 现行有效
---

# Changelog

> Alfred 设计演进的逐版记录。每一条决策都链接到对应的 ADR 与设计会话存档。
> 完整讨论背景见 [design-sessions/2026-08-03-v1-redesign.md](./design-sessions/2026-08-03-v1-redesign.md)。

---

## 2026-08-18 · 文档一致性收敛与 PRD 治理

- [STYLE](./STYLE.md) §2 元信息块由单行｜改为多行 bullets；自身 frontmatter 补 `updated`。
- [VISION](./VISION.md) 补 frontmatter（updated/scope/related）；§2 七条 V1 能力统一「能力—PRD 工作流」格式（去掉孤立加粗的「投递追踪」）；§4 加固约束级边界；元信息「最后更新」同步。
- [PRD：job-seeking](./prd/job-seeking.md) 新增 NFR-4（缺失信息·人工补齐优先）并补工作流二 step 5(d)；工作流一补「缺失補全闭环」并修正子项层级。**工作流二**：修正触发条件（紧接工作流一 → 「同一会话中用户继续投递该岗位」，明确仅针对刚才那个岗位）；推荐改为可迭代反馈闭环；文件交付明确「可直接下载的文件（即下载链接）」。
- [UPLOAD_WORKFLOW](./UPLOAD_WORKFLOW.md) / [GRAPH](./GRAPH.md) / [DATA_MODEL_V2](./DATA_MODEL_V2.md) / [design-sessions](./design-sessions/2026-08-03-v1-redesign.md) / [DATA_MODEL_V3](./DATA_MODEL_V3.md) 元信息块统一「每项一行」渲染修复并同步 `最后更新`。
- [VISION](./VISION.md) 北极星句由裸引用块改为 `[!ABSTRACT]` callout，与元信息块明确区分（纠正「位置误当元信息」）。
- [PRD：job-seeking](./prd/job-seeking.md) 工作流一～多处的 `(a)(b)(c)` 子项改为嵌套有序列表（`a. b. c.`），每个另起一行、渲染清晰。**工作流三**：step2 补「官网投递可能无邮件」说明；投递台账与额外处理改每项一行的嵌套有序列表（mermaid 同步）。
- [LATEST](./LATEST.md) 补 2026-08-18 索引（`updated` 一并补），落实「每次更新自动同步元数据 + CHANGELOG + LATEST」纪律。

## 2026-08-07 · 站点导航去除 status 徽章「!」
- 根因：MkDocs Material 将 frontmatter 的 `status`（如 `现行有效` / `草稿` / `accepted`）当作内置「页面状态」徽章渲染到左侧导航，未知状态值回退成「!」图标。该 `status` 字段是站点治理元数据（见 [STYLE §6.3](../STYLE.md)），供 AI 过滤，不应被主题渲染。
- 修法：新增 [docs/css/extra.css](./css/extra.css) 对 `.md-status` 设 `display:none`，并在 [mkdocs.yml](../mkdocs.yml) 注册 `extra_css`；仅隐藏视觉标记，保留文档内 `status` 数据本身。不影响任何正文内容。

## 2026-08-06 · 文档一致性修正（mermaid / 流程图 / 索引）
- [PRD：job-seeking](./prd/job-seeking.md) 修复工作流三 mermaid 语法错误（节点标签内多余引号导致无法渲染）；为原缺流程图的工作流八（笔试）、十一（Offer 比较）、十四（Coffee Chat）补齐 flowchart，统一为带引号标签风格。
- [LATEST](./LATEST.md) 补 2026-08-06 更新索引（此前停在 08-05）。
- [prd/README](./prd/README.md) 登记求职助理用户流程 PRD（此前误写「暂无特性级 PRD」）。
- [GLOSSARY](./GLOSSARY.md) / [index](./index.md) 补求职领域细化表与 DATA_MODEL_V3 链接。
- 说明：CHANGELOG 此前已含 08-06「工作流十一～十五」条目；DATA_MODEL_V3 仍为草稿（工作流 11/13/14 的结构化字段、activity_log 物化、person 画像隔离待评审）。

## 2026-08-06 · 求职工作流补全（十一～十五）与数据模型细化
- [PRD：job-seeking](./prd/job-seeking.md) 新增工作流十一～十五（Offer 比较 / 静默期保温 / 统计与分析 / Coffee Chat / 合同审查）+ FR-18～FR-22；候选工作流章节裁剪已转正项。
- [DATA_MODEL_V3.md](./DATA_MODEL_V3.md) 扩展：`follow_up.kind`（thank_you / result_follow_up / silence_warmup）、`activity_log`、`contract_review`、`person`/`interaction` 画像字段；补工作流 11–15 数据流行与关键不变量（共 14 条）。

## 2026-08-05 · 求职工作流补全（六～十）与 ER 设计
- [PRD：job-seeking](./prd/job-seeking.md) 新增工作流六～十（面试复盘 / 跟进生成 / 笔试 / Offer / 拒信）+ FR-13～FR-17。
- 产出 [DATA_MODEL_V3.md](./DATA_MODEL_V3.md)（ER 图 + 数据流 + 新表规格 + 不变量草稿），登记 mkdocs nav；并修订 mkdocs.yml 注册。

## 2026-08-03 · V1 重新设计（重大架构定型）

本轮把分散在聊天记录里的概念清单与已落地代码做了对齐，拷问并拍板了 11 个互相锁死的建模决策。全部为 `accepted`。

### 核心结论

| 主题 | 决策 | ADR |
|---|---|---|
| 命名 | `job_posting`(实例) + `occupation`(大类)，退役 `position` | [0009](./adr/0009-naming-job-posting-occupation.md) |
| 真相模型 | `event`(行为真相) 与 `fact`(状态真相) **并列互补**，非派生；`application`/`interview` 独立强类型表 | [0010](./adr/0010-event-fact-dual-truth.md) |
| 多用户 | 私有表带 `user_id`，共享实体/词表（`user`/`person`/`organization`/`occupation`/`skill`/`location`）不带 | [0011](./adr/0011-multi-tenant-boundary.md) |
| 编排框架 | **LangGraph 编排 + Action 执行**双层，不用 OpenAI Agent SDK | [0012](./adr/0012-orchestration-langgraph.md) |
| 上下文 | 常驻单 `chat_thread` + AI 自动 `chat_segment` 分段 + 人工干预 | [0013](./adr/0013-context-segments.md) |
| 简历溯源 | 软引用 + 渲染强制 footnote + 校验阻断无源 | [0014](./adr/0014-resume-provenance.md) |
| 地点 | `location` 实体 + `location_alias` 反查 | [0015](./adr/0015-location-entity.md) |
| 内核边界 | 领域无关内核 + 领域包（V1 求职 / 未来健康·记账） | [0016](./adr/0016-domain-agnostic-kernel.md) |
| 组织泛化 | `company` → `organization`（kind 枚举） | [0017](./adr/0017-organization-generalization.md) |

### 关键不变量（写代码前必须记住）

1. **双真相**：行为写 `event`（append-only），状态物化到 `fact`（带 `valid_from/valid_to`）；二者经 `fact.source_event_id` 双向可追。
2. **Action 唯一写入口**：LangGraph 节点落库一律走 `ActionExecutor`，保留 `dry_run/commit` 与 Fake-provider 可测。
3. **租户过滤**：所有私有表查询带 `user_id`；共享表天然跨用户。
4. **强类型优先**：可归类数据进强类型表，`memory` 仅收容残余。
5. **简历可解释**：bullet 须软引用 `fact`，渲染加 footnote，校验阻断无源。

### 影响面（实现期要做的）

- 重命名迁移：`position → job_posting`、`company → organization`、`position.job_family → occupation`、`tag.namespace='skill' → skill+user_skill`。
- 新增表：`fact`、`chat_segment`、`location`、`location_alias`、`nudge`、`nudge_rule`、`document_version_fact`、`position_requirement`、`experience_source`、`user_skill`（已有则需调整语义）、`interaction`（替代窄 `coffee_chat`）。
- 旧 `event` 语义变更：从"时间轴 kind 表"改为"行为真相日志"；coffee chat / interview 等内容迁移到 `interaction` / `interview` 强类型表。

---

## 2026-08-04 · 个人 OS 扩展与引擎术语重构（重大范围扩展）

把 Alfred 从「求职助理」推进为「个人 OS」。本轮只更新设计文档，不改代码（代码重构在后续实现期）。决策背景见设计会话（brainstorming → domain-modeling → grill-with-docs → to-spec）。

### 核心结论

| 主题 | 决策 | ADR |
|---|---|---|
| 引擎术语 | "Skill" 一词双义 → 改名 **Node**（处理单元）/ **Graph**（LangGraph 真控制流）/ **Action**（原子写原语不变） | [0018](./adr/0018-engine-terminology-node-graph-action.md) |
| Graph 形态 | **Graph 是 Python 代码**（Graph-as-Code）；YAML 只做 Node 配置，不承载控制流 | [0019](./adr/0019-graph-as-python.md) |
| 输入分发 | **四层分发 L0–L3**：模态 → 声明式预筛 → LLM 模糊回路 → 回流；跨域 subgraph 常驻 | [0020](./adr/0020-four-layer-dispatch.md) |
| 领域扩展 | **共享基座 + 领域 Graph 包**：job（原样保留）+ habit / media / progress(+travel) | [0021](./adr/0021-domain-graph-packages.md) |

### 关键变化（文档层）

- 词汇表 [GLOSSARY.md](./GLOSSARY.md)：消除 "Skill" 歧义，引入 Node / Graph / Action、4 层分发、共享基座；旧 ADR 的 "Skill" 一律按 Node 理解。
- 入口 [CONTEXT.md](https://github.com/Gavin-OP/alfred-api/blob/main/CONTEXT.md)：引擎描述从 "Skills" 改为 "Node / Graph / Action"。
- 引擎规范 [GRAPH.md](./GRAPH.md)（新建）：Node 配置 YAML + Graph Python + Action + HITL 状态机。
- 捕获工作流 [UPLOAD_WORKFLOW.md](./UPLOAD_WORKFLOW.md)（新建）：4 层分发 L0–L3 + Mode A/B。
- 数据模型 [DATA_MODEL_V2.md](./DATA_MODEL_V2.md)：新增 habit / media / progress(+travel) 三领域包与 ER 图。
- 愿景 [VISION.md](./VISION.md)：V3 个人管家领域落地为 job/habit/media/progress；前端演进到网页 → APP → 硬件。

### 命名冲突消除

`skill`（小写，领域数据）= `skill` 能力词表 / `user_skill` / `position_requirement`；引擎单元 = **Node**。二者不再混用。

---

## 历史（原型期）

- 0001–0008 见 [adr/README.md](./adr/README.md)：PostgreSQL 选型、前后端分仓、Taro 三端、可带脚本的 Skill 目录、长输入分块、延后 LaTeX 编译、命名 Alfred、services 单一写入口。
