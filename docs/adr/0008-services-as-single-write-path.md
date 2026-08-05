# ADR-0008 · services 层作为唯一落库入口

- 状态：accepted
- 日期：2026-08-01

## 背景（Context）

系统有**两类写入来源**：

1. **人工**：前端调 REST（`POST /v1/companies`、`PATCH /v1/positions/{id}`）
2. **AI**：Skill 解析用户的语音/截图后自动建实体

如果这两条路各写各的（路由里一套 SQLAlchemy 操作、skill 里另一套），必然出现漂移：

- 路由里做了公司名去重，skill 里忘了 → 数据库里三个"字节跳动"
- 路由校验了 `relation` 在受控词表内，skill 没校验 → 图里出现语义碎片
- 改了必填字段，只改了一边

更糟的是，AI 写入是**高频且不可预测**的，脏数据一旦进来，靠人工清理成本极高。

用户对后端的核心要求正是 **"非常 structured 且 robust，能让 AI agents 去调用"**。

## 决策（Decision）

**`app/services/` 是唯一被允许写数据库的地方。**

- services 暴露一组**Action 原语**：`upsert_company` / `upsert_position` / `upsert_person` / `create_note` / `create_event` / `create_reminder` / `add_tags` / `link` / `update_entity` / …
- 每个原语的入参是一个 **Pydantic 模型**（类型校验、枚举校验、受控词表校验都在这里）
- **REST 路由**：解析 HTTP → 调原语 → 序列化响应。路由里不出现 ORM 写操作
- **Skill**：产出 `SkillResult`（一个 Action 列表）→ 框架在**同一事务**内依次执行这些原语
- Skill 保留 `ctx.services` 逃生舱，但文档标注为"绕过审计，非必要不用"

```
REST 路由 ─┐
           ├─→ services（Action 原语 · Pydantic 校验 · 去重 · 事务）─→ PostgreSQL
Skill ─────┘
```

## 理由与代价（Consequences）

**得到**

- **零漂移**：去重规则、状态机校验、受控词表只有一份实现
- **可审计**：每个 Action 连同来源 skill / chunk 写入 `ingest_run`
- **可 dry-run**：`POST /v1/ingest?dry_run=true` 只返回将执行的 Action 列表，不落库——写 skill 时的调试利器
- **可事务**：一次 ingest 的全部 Action 要么都成功、要么全回滚，不会留下"建了公司没建岗位"的半截数据
- **可重放**：解析结果已结构化保存，改了落库逻辑可以重跑历史输入
- **可测试**：services 层单测覆盖全部写路径，skill 测试只需断言"产出了正确的 Action"，不必碰数据库

**代价**

- 多一层间接：加一个字段要同时改 SQLModel、Pydantic 入参、原语实现
- Skill 不能"随手写库"，要按原语表达意图（这正是我们想要的约束）
- 原语表需要维护和文档化

## 备选方案（Alternatives）

| 方案 | 为何不选 |
|---|---|
| **路由和 skill 各自直接用 ORM** | 必然漂移；AI 高频写入下脏数据不可控 |
| **Skill 通过 HTTP 回调自己的 REST API** | 天然共用逻辑，但引入自调用的网络开销、事务无法跨请求、失败处理复杂 |
| **Skill 直接调 services（不经 Action 中间层）** | 共用了逻辑，但失去 dry-run、审计、统一事务与重放能力——而这些恰恰是"robust"的核心 |
| **数据库层用触发器/约束兜底** | 只能兜住格式，兜不住语义（如公司名去重、状态机跃迁） |
