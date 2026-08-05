# ADR-0010 · Event 与 Fact 的并列真相模型

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §6.2](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

用户最初把 `Event` 描述为"核心枢纽，连接 Fact/Person/Organization/Jobs"。若照做，`application` 的 12 态状态机、外键、投递凭证都要塞进 JSONB —— 不能约束、不能索引、不能 JOIN。
但用户又要"给我看 7 月发生的所有事"这种统一时间轴。
调研时我曾提议"Event 是派生的时间轴投影"，被用户否决。

## 决策（Decision）

**两套真相，并列互补，不是谁派生谁：**

- **`event` = 行为的真相（Source of Truth for Behavior）**：append-only 不可变日志，记录"发生了什么"，不记录"现在是什么状态"。
- **`fact` = 状态的真相（Source of Truth for State）**：从 event + 业务规则物化出来的快照，带 `valid_from → valid_to`（纠错时旧值封口、新值开区间），记录"现在/某时段是什么样"。
- 两者通过 `fact.source_event_id` 双向可追。
- **`application` / `interview` / `interaction` 各自是强类型表，不是 Event。** Event 只记录"Application Submitted 这个行为"，细节仍在 `application`。

## 理由与代价（Consequences）

- ✅ 既保住状态机 / 外键 / 索引约束，又能回答"7 月发生了什么"。
- ✅ 天然领域无关（event 只记行为、fact 只记状态），直接服务 V3 管家愿景。
- ❌ 写入需双写（event + fact），需保证一致性（同一事务）。
- ❌ `fact` 需物化逻辑（从 event 推导当前状态）。

## 备选方案（Alternatives）

- **Event 派生投影（被否决）**：用户明确要"行为真相"，不要派生索引。
- **Event 单表继承、是唯一真相**：状态机 / 外键 / 索引全部失效。
- **砍掉 event 用 UNION 视图**：跨类型排序分页性能差，新增类型要改视图。
