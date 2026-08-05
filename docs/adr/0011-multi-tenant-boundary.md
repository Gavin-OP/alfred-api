# ADR-0011 · 多用户边界：私有表带 user_id，共享表不带

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §6.4](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

现状 `user_skill.user_id` 是裸字符串，没有 `user` 表、没有认证、没有隔离。一旦真多人用：
`company.slug`、`position.fingerprint`、`skill.slug` 的唯一约束全部失效；每个查询都要加租户过滤，漏一个就是串号级事故。
事后补租户列是数据库改造里最贵的一类。

## 决策（Decision）

- **私有表**（「我」的活动与状态）每张带 `user_id` / `owner_id`：`conversation`、`chat_message`、`chat_segment`、`event`、`fact`、`application`、`interview`、`experience`、`experience_source`、`user_skill`、`document_version`、`document_version_experience`、`document_version_fact`、`resume_asset`、`note`、`memory`、`reminder`、`nudge`。
- **共享表**（跨用户的真实世界实体 / 词表）**不带** `user_id`：`user`(账户)、`person`、`organization`、`occupation`、`skill`、`skill_alias`、`location`、`location_alias`、`job_posting`。
- 认证后置：现在先加 `user_id` 列 + 复合唯一约束（`UNIQUE(user_id, slug)`），用固定默认用户，暂不接登录。

## 理由与代价（Consequences）

- ✅ 现在加列几乎零成本；共享词表天然跨用户复用。
- ✅ 明确边界，避免"哪些表要过滤"的歧义。
- ❌ 每个私有表查询都要带 `user_id` 过滤（用 repository 基类统一处理）。

## 备选方案（Alternatives）

- **完全单用户，将来整库迁移**：省当下事，但那次迁移要动每张表、每个查询、每个唯一约束。
- **一人一库（DB-per-user）**：隔离最彻底，但迁移要对 N 个库执行，跨用户共享技能词表无处安放。
