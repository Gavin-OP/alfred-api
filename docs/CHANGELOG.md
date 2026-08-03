# Changelog

> Alfred 设计演进的逐版记录。每一条决策都链接到对应的 ADR 与设计会话存档。
> 完整讨论背景见 [design-sessions/2026-08-03-v1-redesign.md](./design-sessions/2026-08-03-v1-redesign.md)。

---

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

## 历史（原型期）

- 0001–0008 见 [adr/README.md](./adr/README.md)：PostgreSQL 选型、前后端分仓、Taro 三端、可带脚本的 Skill 目录、长输入分块、延后 LaTeX 编译、命名 Alfred、services 单一写入口。
