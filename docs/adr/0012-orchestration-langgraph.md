# ADR-0012 · 编排框架：LangGraph 编排 + Action 执行

- 状态：accepted
- 关联讨论：[design-sessions/2026-08-03-v1-redesign.md §7.4](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

工作流是"用户输入多模态 → AI 解析成结构化 → 用户确认后入库"，本质确定性管道 + 一次人工确认，不是自主 Agent 循环。
但产品要的是**有状态、可中断、可审计、多 Workflow 共存、可绑 PostgreSQL 的「个人 OS」**，不是全自动工具调用 Agent。
现状自研机制（`ingest_request` + `dry_run/commit_ingest`）已实现持久 + HITL，但与"多 workflow 共存 + 复杂状态图"的演进需求耦合松散。

## 决策（Decision）

- **采用 LangGraph（Python）作为主编排骨架**，自研一层极薄的 Agent Runtime 封装作为接口。
- **不用 OpenAI Agent SDK**（偏 OpenAI 绑定、Sessions 无崩溃恢复、HITL 需自写）。
- **关键约束：分层** —— LangGraph 只做**编排层**（图定义 workflow、拥有 checkpoint 与 HITL 中断）；每个需落库的 graph 节点调用现有 **`ActionExecutor`**（单一写入口）。即「LangGraph 管状态与中断，Action 管写库」。
- 现有 `ingest_request` / `dry_run` / `commit_ingest` 机制**保留并下沉为 LangGraph 节点的执行单元**。
- 测试用 LangGraph in-memory checkpointer + 现有 SQLite 内存库，不依赖真实 PG。

## 理由与代价（Consequences）

- ✅ 原生 Postgres checkpointer（持久可恢复）+ interrupt/resume（解析后等确认再落库），契合工作流 (a)(b)(c)。
- ✅ 复用现有 Action 铁律：`Action` 唯一落库入口、`dry_run/commit` 语义、Fake-provider 可测、模型无关。
- ✅ 对用户 Vibe Coding 友好：不自造状态机。
- ❌ 引入 LangGraph 学习 / 调试面。
- ❌ plan 原「自研 Skill+Action 作为唯一编排」表述需重写为「LangGraph 编排 + Action 执行」双层。

## 备选方案（Alternatives）

- **纯自研编排**：要重造 durable checkpoint + interrupt/resume，对 Vibe Coding 不划算（用户已明确不选）。
- **OpenAI Agents SDK**：Sessions 无崩溃恢复、HITL 需自写、偏 OpenAI 绑定，与模型无关原则冲突。
