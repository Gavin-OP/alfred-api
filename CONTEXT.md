# CONTEXT.md — Alfred 项目入口

> 这是项目的**单一上下文入口**。任何 AI / 协作者上手先读这份，再按需下行到：
> - **词汇表（权威）**：[docs/GLOSSARY.md](docs/GLOSSARY.md)
> - **决策记录**：[docs/adr/](docs/adr/)（含 README 索引）
> - **愿景与演进**：[docs/VISION.md](docs/VISION.md)
> - **数据模型**：[docs/DATA_MODEL_V2.md](docs/DATA_MODEL_V2.md)
> - **上传/捕获工作流**：[docs/UPLOAD_WORKFLOW.md](docs/UPLOAD_WORKFLOW.md)
> - **引擎规范（Node/Graph/Action）**：[docs/GRAPH.md](docs/GRAPH.md)
>
> 领域文档维护规则见 `docs/agents/domain.md`（由 `domain-modeling` 技能驱动）。

---

## 1. Purpose（项目是做什么的）

Alfred 是一个**领域无关的「个人 OS / 个人管家」**。从一个求职助理起步，内核保持领域无关，通过**领域包（Domain Graph）**逐步扩展覆盖：求职（job）、习惯（habit）、作品/书影音（media）、进度/项目/旅行（progress）。所有领域共享同一套引擎、同一份共享基座、同一种记忆与上下文。

## 2. Kernel / Domain 边界

- **共享基座（Kernel）**：`person` / `organization` / `occupation` / `skill` / `location` / `event` / `fact` / `chat_*` / `memory` / `attachment` / `reminder` / `nudge`。不属于任何领域，由跨域 subgraph 维护。边界与理由见 [ADR-0016](docs/adr/0016-domain-agnostic-kernel.md)。
- **领域包（Domain Package）**：每个领域是一个 **Domain Graph**，自带私有扩展表，仅在私有表加 `user_id`。首个领域 job 已落地；habit / media / progress 为本轮新增（先定模型，逐步实现）。

## 3. Engine：Node / Graph / Action（三层抽象）

> **本轮术语重构（2026-08-04）**：原 "Skill" 一词既指引擎单元又指能力词表，产生冲突。现引擎单元改名 **Node**，多个 Node 组成 **Graph**，**Action** 是 Node 内部的原子写原语。旧 ADR 中的 "Skill" = 现在的 Node。详见 [ADR-0018](docs/adr/0018-engine-terminology-node-graph-action.md)。

| 层 | 名称 | 角色 | 形态 |
|---|---|---|---|
| 底层 | **Action** | 原子写原语，1:1 映射 `services` 函数，单一落库入口 | `services/executor.py` + `ACTION_KINDS` |
| 中层 | **Node** | 处理单元：LLM 抽取（prompt+output_schema）→ 产出 `out` → 跑一组 Action 改共享 state。**不含控制流** | `alfred/nodes/<name>/node.yaml`（配置）|
| 顶层 | **Graph** | 带共享 state 的 LangGraph `StateGraph`；Node 经 Edge 串联/分支/并行/循环。多个领域 Graph 可封 subgraph 经外层 router 再编排 | `alfred/graphs/<domain>.py`（**Python 声明，不是 YAML**）|

- **Declarative Node**：以 `node.yaml` 为主（`match` / `chunking` / `prompt` / `output_schema`）。
- **Scripted Node**：以 Python 函数体为主，可返回 `Command(goto=...)` 做**数据驱动的动态路由**（静态流程仍在 Graph 里画）。
- **Routing（4 层分发 L0–L3）**：模态 → 声明式预筛 → LLM 模糊回路 → 回流。跨域 `memory`/`capture` subgraph 常驻，任意 Domain Graph 可调用。见 [UPLOAD_WORKFLOW.md](docs/UPLOAD_WORKFLOW.md)。
- **dry_run / commit**：Action 落库前的预览与确认，迁移为 LangGraph `interrupt` / `resume`（HITL），保证脏数据不入库。

## 4. Data Model（数据模型）

归一化优先：凡需跨内部查询 / 联结的，必须关系化，不用 JSON 数组。（[DATA_MODEL_V2.md](docs/DATA_MODEL_V2.md) 第 6 铁律）

- 唯一键：UUID 主键；`slug`（公司）/ `fingerprint`（岗位=sha256(公司id+标准化职位名+地点)）作业务去重。
- 技能三义分离 + 平权存储：`skill` 仅作共享词表；"我的技能"=`user_skill`，"JD 所需"=`position_requirement(kind=skill)`，相似技能用 `skill_alias` 反查。
- 经历库：一条经历一行，仅存原始口语稿 `raw_input_md`；AI 优化稿随岗位进 `document_version_fact`，不建 bullets 表。

## 5. Memory & Context（记忆与上下文）

- `chat_message`（全量双方对话、带时间戳）为底座——你说的每句话都留存、日后能调出。
- `chat_thread` 分组/归档 + `memory` 实体（抽取的结构化事实/偏好/意图/上下文）做长期累积。
- 召回 = BM25 全文 + 向量余弦混合加权（7:3）+ 指数时间衰减 salience；PG 走 `<=>` + tsvector，SQLite 退化为 Python 端余弦 + LIKE（封装在 `MemoryStore`，对调用方只暴露 `recall()`）。

## 6. 词汇摘要（必须一致）

| 词 | 含义 | 易混淆 |
|---|---|---|
| **Node** | 引擎处理单元（原 "Skill"）| 不是 `skill`（能力词表）|
| **Graph** | LangGraph StateGraph，Node 经 Edge 组成 | 不是命名空间 |
| **Action** | Node 内原子写原语 | — |
| **Domain Graph** | 一个领域的 Node+Edge 集合（可复用模块）| — |
| **skill** | 能力词表（领域数据）| 不是引擎 Node |
| **Kernel / 共享基座** | 跨领域表（person/organization/skill/location/...）| 不属于任何 Domain Graph |
| **L0–L3** | 4 层分发：模态/预筛/LLM模糊/回流 | — |

## 7. Dev Workflow（Vibe Coding）

自然语言需求 → `/grilling`（ relentless 访谈定稿，grill-with-docs 同步出 ADR/术语表/CHANGELOG）→ `to-spec` 把 spec 发 GitHub issue tracker（标 `ready-for-agent`）→ Agent 按规格实现 → TDD 验证。

工程技能（Matt Pocock 集）配置在 `docs/agents/`；issue 追踪与 triage 标签见 `docs/agents/issue-tracker.md` 与 `docs/agents/triage-labels.md`。
