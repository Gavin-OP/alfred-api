---
type: reference
status: 现行有效
---

# Alfred 领域词汇表（Glossary / Ubiquitous Language）

> 这是 Alfred 全项目的**唯一权威词汇表**。所有文档、ADR、代码注释、prompt 都必须使用这里的词，避免"同一个意思三种叫法"。
> 改动本表 = 改全项目的通用语言，请走 `domain-modeling` 技能确认。
>
> **本轮重大术语变更（2026-08-04）**：原 "Skill" 一词同时指代「引擎处理单元」与「简历能力词表」，产生命名冲突。现把**引擎单元改名为 Node（节点）**，Graph（图）由多个 Node 经 Edge 组成，Action（动作）是 Node 内部原子写原语。详见 [ADR-0018](./adr/0018-engine-terminology-node-graph-action.md)。**旧 ADR（0001–0017）里出现的 "Skill" 一律指现在的 Node。**

---

## A. 引擎本体（Engine Primitive）—— 三层抽象

| 术语 | 英文 | 定义 | 实现位置 / 形态 |
|---|---|---|---|
| **Action** | Action | 引擎**最底层**的原子写操作原语。1:1 映射到 `services` 里的落库函数。无控制流、无 LLM，只负责把一块数据写进 DB（或 dry-run 预览）。 | `services/executor.py` 的 handler；`ACTION_KINDS` 注册表 |
| **Node** | Node（节点） | 一个**处理单元** = LLM 抽取（`prompt` + `output_schema`）→ 产出结构化 `out` → 跑一组 Action 改共享 state。Node **不含控制流**，控制流在 Graph 里。原 "Skill" 目录。 | `alfred/nodes/<name>/node.yaml`（配置）+ 同名 Python 处理入口 |
| **Graph** | Graph（图） | 带**共享 state** 的 LangGraph `StateGraph`。多个 Node 经 **Edge** 串联 / 分支 / 并行 / 循环；支持 DAG 也能有环。用 **Python 声明**（LangGraph API），**不是 YAML**。 | `alfred/graphs/<domain>.py`（Python 代码）|
| **Domain Graph** | 领域图 | 围绕一个领域（job / habit / media / progress）的一组 Node + 它们之间的 Edge，作为**可复用模块**。 | 每个领域一个 `alfred/graphs/<domain>.py` |
| **Subgraph** | 子图 | 被外层 Graph 包进去复用的 Domain Graph 或跨域 subgraph。 | LangGraph `add_subgraph` |
| **Outer Router** | 外层路由 | 把多个 Domain Graph 编排成总流的顶层 Graph；决策"进哪个领域图"。 | `alfred/graphs/root.py` |
| **Edge** | 边 | Graph 中连接两个 Node 的有向弧，可带条件（conditional edge）实现分支 / 并行 / 循环。 | `StateGraph.add_edge` / `add_conditional_edges` |
| **Manifest** | 清单 | Node 的声明式配置文件（`node.yaml`）。描述：身份（name/domain/version）、路由（match）、分块（chunking）、prompt、output_schema。**只做配置，不做控制流**。 | `alfred/nodes/<name>/node.yaml` |
| **State** | 状态 | LangGraph Graph 在运行期间携带的共享状态（TypedDict）。所有 Node 读它、改它。 | `AlfredState`（见 [GRAPH.md](./GRAPH.md)）|
| **ui_intent** | UI 意图 | Graph 产出的"客户端下一帧该渲染什么"结构（capture 表单 / 确认预览 / 列表 / 多模态提示）。引擎**无头**，不关心客户端形态。 | `AlfredState["ui_intent"]` |

### 引擎三层与旧术语对照
```
旧：Skill（目录，含 prompt + output_schema + actions）
新：Node（处理单元）  =  prompt + output_schema  →  一组 Action
                        多个 Node 经 Edge 组成  →  Graph（Python 声明）
                        Action（原子写原语，不变）
```

### Declarative / Scripted（声明式 / 脚本式）
- **Declarative Node**：以 `node.yaml` 为主、逻辑可数据驱动的 Node（如 `extract_job`）。
- **Scripted Node**：以 Python 函数体为主、含复杂业务逻辑或动态 `Command(goto=...)` 路由的 Node（如 `route_dispatch`）。

### Routing（路由 / 分发）
沿用 `registry.select()` 关键词 / URL / 模态打分。详见 [UPLOAD_WORKFLOW.md](./UPLOAD_WORKFLOW.md) 的 **4 层分发（L0–L3）**。

---

## B. 数据模型（Data Model）

### 1. 共享基座 / 内核（Shared Substrate / Kernel）—— 不属于任何 Domain Graph
> 跨领域常驻，由跨域 subgraph 维护，任意 Domain Graph 可调用。边界见 [ADR-0016](./adr/0016-domain-agnostic-kernel.md)。

| 表 | 含义 |
|---|---|
| `person` | 人（含关系边 `person_relation`）|
| `organization` | 组织（company 泛化，kind 枚举）|
| `occupation` | 职业大类（标准化维度）|
| `skill` | **能力词表**（共享，仅定义一次；"Excel" 只存一行）|
| `location` | 地点实体（职位的地点 / 习惯场所 / 旅行足迹统一挂钩）|
| `location_alias` | 地点反查别名 |
| `event` | 行为真相日志（append-only）|
| `fact` | 状态真相物化（带 valid_from/valid_to）|
| `chat_message` / `chat_thread` | 全量对话与线程 |
| `memory` | 抽取的结构化长期记忆（事实 / 偏好 / 意图 / 上下文）|
| `attachment` | 多模态附件（语音 / 截图 / 文件）|
| `reminder` / `nudge` / `nudge_rule` | 用户手设提醒 / 系统算出提醒 / 提醒规则 |

### 2. 领域包（Domain Packages）—— 每个 Domain Graph 私有的扩展表
> 第一个领域 **job** 已落地；**habit / media / progress** 为规划中的未来领域（先定模型，本轮暂不实现，仅通过领域无关内核保留扩展能力）。

| 领域 | 关键表 | 说明 |
|---|---|---|
| **job**（求职）| `job_posting` `application` `interview` `offer` `experience` `experience_source` `position_requirement` `user_skill` `document_version` `document_version_fact` | 现有表原样保留并包成 Domain Graph |
| **habit**（习惯）| `habit` `habit_log` | 习惯定义 + 打卡记录（`habit_log` 可挂 `location_id`）|
| **media**（作品/书影音）| `work_item` `review` | 书/影/剧/游戏 + 读后感/观后感（`work_item` 可挂 `location_id`）|
| **progress**（进度/项目/旅行）| `project` `progress_log` `trip_day` | `project.kind` 区分 vlog/project/**trip**；旅行按天 `trip_day` 挂 `location_id` |

> 注意：**travel（旅行）归入 progress 领域**（`project.kind='trip'` + `trip_day`），靠共享 `location` 实现"今天去哪、明天去哪"。

### 3. 易混淆词
| 词 | 指什么 | 别和谁混 |
|---|---|---|
| **skill**（小写，领域数据）| `skill` 能力词表，简历/JD 的能力 | 引擎 **Node**（已改名，不再叫 skill）|
| **user_skill** | 我的能力（关联 `skill`）| 引擎 Node |
| **position_requirement** | 岗位所需（kind=skill/qualification/experience）| 引擎 Node |
| **occupation** | 职业大类（跨公司查同职业岗位）| `job_posting`（具体岗位实例）|

---

## C. 记忆与上下文（Memory & Context）
见 [ADR-0013](./adr/0013-context-segments.md) 与记忆系统（chat_message 底座 + memory 实体 + 混合召回）。

---

## D. 流程与方法（Process）
- **dry_run / commit**：Action 落库前的预览与确认，迁移为 LangGraph `interrupt` / `resume`（HITL）。
- **4 层分发 L0–L3**：模态 → 声明式预筛 → LLM 模糊回路 → 回流。见 [UPLOAD_WORKFLOW.md](./UPLOAD_WORKFLOW.md)。
- **Vibe Coding**：开发方式——自然语言需求经 `/grilling` 转规格，spec 发 issue tracker，Agent 按规格实现。

---

## E. 外部 / 项目（Project）
- **Alfred**：项目名（[ADR-0007](./adr/0007-naming-alfred.md)）。
- **alfred-api**：后端仓库（本仓库）；**alfred-web**：前端仓库（小程序→网页→APP→硬件）。
- **Matt Pocock 工程技能**：`to-tickets` / `triage` / `to-spec` / `grill-with-docs` / `domain-modeling` 等，配置见 `docs/agents/`。

---

## F. 文档类型（Document Types）

本站文档按 [ADR-0022](./adr/0022-documentation-taxonomy-prd-design-adr.md) 分为以下类型，`type` 以 YAML frontmatter 为准：

| 术语 | 英文 | 定义 | 落地位置 |
|---|---|---|---|
| **PRD** | Product Requirements Document | 产品需求文档：关注"做什么"——功能、场景、业务需求、成功指标 | `docs/prd/` |
| **Design Document** | 设计文档 | 设计文档：关注"怎么做"——ER 图、数据流图（DFD）、接口、权衡 | `docs/` 设计文档分区 |
| **ADR** | Architecture Decision Record | 架构决策记录：关注"为什么这样决策"，可标记取代旧 ADR | `docs/adr/` |
| **Specification** | 规范 | 统称：PRD + Design + ADR 构成的可回溯规范集合；用户每次新意见都是对 specification 的更改 | — |
| **VISION** | 北极星 | 最高层产品方向，为 PRD 提供方向，本身不是 PRD | `docs/VISION.md` |
