# 术语表（Glossary）

**Glossary = 项目里反复出现的专有名词的统一定义。**

它的作用很实际：当你、我、或未来任何一个 AI 协作者说 "skill" 时，指的必须是同一个东西。术语一旦各说各话，设计讨论就会空转。

> 约定：本文件是术语的**唯一权威定义**。代码、注释、文档、commit message 都以此为准。新增术语请在此登记。

---

## A. 架构与流程

### Ingest（摄入）
把**原始输入**（文字、语音、链接、截图）交给后端解析成结构化数据的**整个过程**。
对应端点 `POST /v1/ingest`。这是用户和 Alfred 交互的唯一主入口。
> 中文可读作"摄入"或"投喂"。别和 "import（导入现成数据）" 混用。

### Normalizer（归一化器）
Ingest 的第一步。把四种模态统一变成**纯文本**：语音走 STT、截图走视觉描述/OCR、链接走正文抽取、文字直接透传。
它的产物是 `SkillInput`。

### Skill（技能）
**一个目录**，代表 Alfred 会做的一件事（拆职位、记笔记、整理 coffee chat…）。
包含 `skill.yaml`（说明书，必需）和可选的 `run.py`（脚本）。
**用户可以自己写、自己加**，加技能不需要改后端代码。旧版 Skills 规范见 [SKILLS.md](archive/v0.1-baseline/SKILLS.md)（已作废）。
> 注意：这里的 skill 指"AI 技能包"，**不是**简历里的"技能"。后者在本项目里是 `skill`（受控词表）+ `user_skill`（我的技能），见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)。

### Manifest（技能清单）
即 `skill.yaml` 的内容，经 Pydantic 模型 `SkillManifest` 校验后的对象。包含身份、路由规则、分块策略、prompt 模板、输出结构、动作列表。

### Declarative Skill / Scripted Skill（声明式 / 脚本式技能）
- **声明式**：只有 `skill.yaml`，走框架的通用流水线。适合"一次 AI 调用就够"的任务。
- **脚本式**：另有 `run.py`，框架调用其 `execute(ctx, inp)`。适合需要控制流的复杂任务。

### Routing（技能路由）
决定这次输入该用哪个 skill 的过程。顺序：显式指定 → 关键词/URL 预筛 → LLM 路由 → 兜底 `capture_note`。

### Chunking（分块）
输入过长时，把文本切成若干段分别调用 AI 的策略。
- `none`：不切，单步调用
- `fixed`：按字符数切，段间留重叠
- `topic`：先让 AI 划分主题边界再切

**分块由 Skill 自行决定，对外接口不变**（见 [ADR-0005](./adr/0005-chunked-pipeline-for-long-input.md)）。

### Reduce（归并）
分块流水线的可选收尾步骤：把各段的抽取结果再交给 AI 做一次全局总结/去重。

### Action（动作原语）
Skill 唯一被允许产生的"副作用"，比如 `upsert_company`、`create_note`、`link`。
每个 Action 一一对应 `app/services/` 里的一个函数，**与 REST 路由共用同一实现**。
Skill 返回的是 Action 列表，由框架统一在一个事务里执行（见 [ADR-0008](./adr/0008-services-as-single-write-path.md)）。

### SkillContext (`ctx`)
框架传给脚本式 skill 的工具箱：`ctx.llm` / `ctx.stt` / `ctx.vision` / `ctx.manifest` / `ctx.render()` / `ctx.chunk()` / `ctx.merge()` / `ctx.services`（逃生舱）。

### SkillResult
Skill 的返回值：`actions[]`（要执行什么）+ `summary_md`（回给用户的话）+ `warnings[]`。

### Dry run（空跑）
`POST /v1/ingest?dry_run=true`：完整跑一遍解析，但**只返回将要执行的 Action，不写数据库**。调试技能时用。

### Provider（供应商适配器）
对接外部 AI 服务的适配层，把不同厂商的差异藏在统一接口后面。
三类：`LLMProvider`（文本/JSON）、`VisionProvider`（看图）、`STTProvider`（语音转文字）。
实现包括 `OpenAICompatibleProvider`、`GeminiProvider`、`WhisperSTTProvider`，以及测试专用的 **`FakeProvider`**。

### Fallback（降级 / 回退）
AI 不可用或多次返回不合规结构时的兜底：用规则（正则、启发式）抽取最低限度信息，**至少把原始输入存成一条 note + attachment**。
原则：**输入永不丢失**。

---

## B. 数据模型

> ⚠️ **本节约一半概念来自 v0.1 基线。** 当前权威数据模型见 [`DATA_MODEL_V2`](./DATA_MODEL_V2.md) 与 §B2；其中 `entity_link`、`tag.namespace='skill'`、`position.job_family` 在 V2 已**移除**（见各条内 ⚠️ 标注与 §E）。

### Entity（实体）
数据库里的一个"东西"，即网络图中的**节点**。v0.1 原列 13 类：`company` `position` `person` `application` `interview` `offer` `document_asset` `document_version` `note` `event` `reminder` `tag` `attachment`——其中 **`company`→`organization`**、**`position`→`job_posting`**、技能归类用的 **`tag`→`skill`** 已在 V2 更名/移除（详见 [`DATA_MODEL_V2`](./DATA_MODEL_V2.md)）。

### EntityRef（实体引用）
指向某个实体的轻量指针：`{type, id}`（列表场景会附带 `label`、`snippet` 等展示字段）。
用于跨类型引用，比如 `entity_link` 的两端、`reminder.target`、查询结果。

### EntityLink（通用边）
连接**任意两个实体**的关系记录：`(source_type, source_id) --relation--> (target_type, target_id)`。
这是"任何东西都能当 filter"的底层支撑——一条笔记可以同时链到公司、岗位、某个人和某项技能，而不需要为每种组合建表。

> ⚠️ **V2 已移除 `entity_link`。** 强关系一律走外键强类型表（`organization`→`job_posting`→`application`→`interview`/`offer`）；弱关系走受控关系词 + 专门关系表（如 `interaction_person`、`interview_interviewer`、`application_evidence`）。不再有任意两实体自由连线的一张总表。

### Relation（关系词）
`entity_link.relation` 的值，来自**受控词表**（`mentions` / `about` / `works_at` / `referred_by` / `interviewed_by` / `participant` / `evidence` / `derived_from` / `requires_skill` / `similar_to` / `related_to` / …）。
**AI 不允许发明新 relation**，未登记的会被拒绝并降级为 `related_to`。
> 为什么要管控：如果 AI 每次自由发挥（"提到过"/"相关"/"关于"/"涉及"），图的语义会碎成一地，没法查询。

### Strong relation vs Weak relation（强关系 / 弱关系）
- **强关系**用外键：Company → Position → Application → Interview/Offer。有引用完整性保证。
- **弱关系**用 `entity_link`：任意实体间的自由关联。

### Tag / Namespace（标签 / 命名空间）
`tag` 通过 `entity_tag` 挂到任意实体上。`namespace` 区分标签的种类（`skill` / `industry` / `job_family` / `topic` / `emotion` / `custom`），避免"Python（技能）"和"焦虑（情绪）"混在同一个列表里。

> ⚠️ **V2 已移除 `tag.namespace='skill'`。** 技能归类改由共享词表 `skill`（+ `user_skill` / `position_requirement`）实现（见 [`DATA_MODEL_V2`](./DATA_MODEL_V2.md)）。`tag` 作为自由标签的兜底能力保留，但不再承担"技能"语义。

### Job family（职位类别）⚠️ 已废弃
旧 `position.job_family`（`swe`/`quant`/`data`/`pm`/`research`/…）已在 V2 被 **`occupation`** 取代（见 [ADR-0009](./adr/0009-naming-job-posting-occupation.md)）。"某一类职位"作为 filter 现在由 `occupation` 实现（§E 已废弃列表另有说明）。

### Slug
人类可读的稳定唯一标识，如 `bytedance`、`jane-chen-hsbc`。
用户要求"每个人都要有 unique identifier"→ 即 `person.slug`。

### Fingerprint（指纹）
`position.fingerprint`，由 `(company_id, 归一化标题, 地点)` 哈希而来，用于**同一岗位从不同渠道被多次录入时的去重**。

### Upsert
"存在就更新，不存在就创建"。Skill 反复摄入同一家公司/同一个岗位时，靠 upsert + 去重键避免产生重复行。

### Soft delete（软删除）
删除只是给 `archived_at` 打时间戳，数据仍在。查询默认过滤。`?hard=true` 才物理删除。

### Origin / Confidence（溯源列）
`origin` 记录这行数据是 `manual`（我填的）还是 `skill`/`ingest`（AI 填的）；`confidence` 是 AI 的置信度，分块合并冲突时取高者。
> 作用：永远分得清"哪些是我确认过的事实，哪些是 AI 猜的"。

### IngestRequest / IngestRun（摄入请求 / 摄入运行）
- `ingest_request`：一次用户输入的记录（原文、链接、选中的 skill、状态、给用户的回复）
- `ingest_run`：**每一次 AI 调用**的记录（阶段、chunk 序号、模型、token、耗时、原始响应、解析结果、错误）

有了它们，AI 解析从黑盒变成可重试、可审计、可算成本、可 debug 的过程。

### Funnel（漏斗）
投递指标统计：投递数 → 回复数 → 面试数 → Offer 数，及各级转化率。端点 `GET /v1/stats/funnel`。

### Ghosting（已读不回）
投了简历后公司长期无任何回应。对应 `application.status = ghosted`。

---

## B2. V1 重新设计新增术语（2026-08-03）

> 以下术语来自本轮重新设计，与 [design-sessions/2026-08-03-v1-redesign.md](./design-sessions/2026-08-03-v1-redesign.md) 及 ADR-0009~0017 配套。旧术语见 §E 已废弃列表。

### job_posting（招聘实例）
具体的"一次招聘"：某公司 + 某 `occupation` + 某 `location`，带唯一 `fingerprint` / UUID。替代旧 `position`。见 [ADR-0009](./adr/0009-naming-job-posting-occupation.md)。

### occupation（职位大类）
长期稳定、跨公司可比的职位类别（software engineer / data scientist / psychologist）。替代旧 `position.job_family`。

### organization（组织）
泛化的"机构"：`company` / `school` / `club` / `ngo` / `government` / `fund` / `open_source_office` / `other`。替代旧 `company`。见 [ADR-0017](./adr/0017-organization-generalization.md)。

### fact（状态真相）
从 event + 业务规则**物化**出来的当前状态快照，带 `valid_from → valid_to`。是结构化数据的真相源（"会 Excel at high" 存于此）。与 `event` 并列互补，不是派生关系。见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)。

### event（行为真相）
**不可变行为日志**（append-only），记录"发生了什么"，不记录状态。如 `FACT_DECLARED` / `APPLICATION_SUBMITTED`。领域无关。与 `fact` 并列。见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)。
> ⚠️ 注意：这与旧 `event`（带 `kind` 的时间轴表，如 `coffee_chat`）**语义不同**。新版 event 是行为真相，求职事件（coffee chat / interview）改为 `interaction` / `interview` 强类型表，其"发生"行为才写 event。

### chat_segment（对话段落）
`chat_thread` 内的语义段落；AI 检测主题漂移自动开新段，用户可重命名/合并/归档。见 [ADR-0013](./adr/0013-context-segments.md)。

### nudge / nudge_rule（主动提醒）
`nudge_rule` = 系统主动提醒的规则；`nudge` = 据此生成的提醒实例。与用户手设的 `reminder` **分表不混**。主推送通道 = 后端邮件 + 每日聚合。

### memory（残余记忆）
无法归入强类型表的非结构化内容（kind = `fact`/`preference`/`intent`/`context`）。"强类型表为真相，memory 仅收容残余"。

### skill（共享技能词表）
受控技能词表（"Excel" 全库只定义一次），区别于"AI 技能包"（Skill 目录，见 §A）。三种"技能"严格区分：AI 技能包 / 我的技能（`user_skill`）/ JD 所需技能（`position_requirement`）。

### user_skill（我的技能）
`skill` 的私有投影（带 `proficiency` 1–5），是技能匹配查询的 JOIN 主体。与 `fact` 中"技能声明"保持同步。

### location / location_alias（地点实体 + 别名）
`location`：结构化地点（country/city/district/admin_code）；`location_alias`：别名反查（香港/HK/Hong Kong → 同一行）。`job_posting.location_id` 指向它。见 [ADR-0015](./adr/0015-location-entity.md)。

### document_version_fact（简历溯源 link）
简历 bullet → `fact` 的软引用多对多 link，支撑渲染 footnote 与校验阻断无源。见 [ADR-0014](./adr/0014-resume-provenance.md)。

---

## C. 求职领域

| 术语 | 含义 | 落点 |
|---|---|---|
| **CV / Resume** | 简历 | `document_asset.kind = resume` |
| **CL / Cover Letter** | 求职信 | `document_asset.kind = cover_letter` |
| **Version（版本）** | 一次不可变的文档源码快照 | `document_version` |
| **Tailor（定制）** | 针对某个具体岗位修改简历/CL | `POST /v1/documents/{id}/tailor` |
| **Application（投递）** | 一次具体的投递行为 | `application` |
| **OA** | Online Assessment，线上笔试/测评 | `interview.kind = oa` |
| **Referral（内推）** | 通过熟人推荐投递 | `application.referrer_person_id` |
| **Recruiter（猎头/招聘官）** | | `person.roles` 含 `recruiter` |
| **Coffee chat** | 非正式的信息交流会面 | `interaction.kind = coffee_chat`（原 `event.kind`，见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)） |
| **Follow-up（跟进）** | 投递/面试后的主动联络 | `next_followup_at` + `reminder` |
| **Visa sponsorship（签证担保）** | 公司是否为外籍员工办工签 | `position.visa_sponsorship` |
| **Work mode** | onsite / hybrid / remote | `position.work_mode` |
| **Seniority（资历层级）** | intern / new_grad / junior / mid / senior / lead | `position.seniority` |

---

## D. 工程与流程

### MVP
**Minimum Viable Product，最小可行产品**。
不追求一步到位做完整系统，而是先做一个**能跑起来、能真正用**的最小版本，解决核心痛点后再逐步扩展。
本项目的 MVP = **后端 DB 与 API 全套做扎实**，前端只做最小调用界面。

### ADR
**Architecture Decision Record，架构决策记录**。
记录"我们为什么这么决定"，而不只是"现在是怎样"。见 [adr/](./adr/README.md)。

### Milestone（里程碑）
一组构成可交付版本的工作集合，如 `v0.1-ingest`。在 GitHub 上对应一个 Milestone，下挂若干 Issue。旧版路线图见 [ROADMAP.md](archive/v0.1-baseline/ROADMAP.md)（已作废）。

### TDD
**Test-Driven Development，测试驱动开发**。先写会失败的测试（红），再写让它通过的最小实现（绿），最后重构。
本项目铁律：**任何测试都不得依赖真实 API key 或外网**（靠 `FakeProvider`）。

### Seam（接缝）
模块之间的边界。好的接缝是"窄接口 + 宽实现"——对外暴露的方法很少，内部隐藏大量复杂度。
本项目的关键接缝：`services` 的 Action 原语、`providers` 的 Protocol、`skills` 的 `execute()`。

### MCP
**Model Context Protocol**。让 AI 工具连接外部系统的协议。本项目用 **GitHub MCP** 来创建和跟踪 milestone / issue / label。

### OpenAPI
FastAPI 自动生成的 API 契约描述（`/openapi.json`）。前端用 `openapi-typescript` 由它生成 TypeScript 类型，**取代手写共享类型包**。

---

## E. 已废弃的术语

| 术语 | 状态 | 说明 |
|---|---|---|
| `claw` / `@claw/*` | ❌ 废弃 | 旧脚手架自动生成的 npm 作用域前缀，无业务含义。见 [ADR-0007](./adr/0007-naming-alfred.md) |
| `jobhunter` | ⚠️ 仅指旧原型 | 指代旧的 Node/Express/Prisma/SQLite 仓库（已归档只读）。新项目一律称 **Alfred** |
| `@claw/shared` | ❌ 废弃 | 手写共享类型包，被 OpenAPI 自动生成取代 |
| `{ok, data, error}` 信封 | ❌ 废弃 | 旧后端的响应格式。Alfred 直接返回资源对象，错误用 HTTP 状态码 + `detail` |
| `position` | ❌ 废弃 | 旧"具体招聘实例"表。改为 **`job_posting`**（见 [ADR-0009](./adr/0009-naming-job-posting-occupation.md)） |
| `company` | ❌ 废弃 | 旧"公司"表。改为 **`organization`**（见 [ADR-0017](./adr/0017-organization-generalization.md)） |
| `job_family` | ❌ 废弃 | 旧 `position.job_family`。改为 **`occupation`**（见 [ADR-0009](./adr/0009-naming-job-posting-occupation.md)） |
| `tag.namespace = 'skill'` | ⚠️ 废弃 | 旧"技能标签"方案。改为共享 **`skill`** 表 + 私有 **`user_skill`**（见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)、[ADR-0011](./adr/0011-multi-tenant-boundary.md)） |
| 旧 `event`（时间轴 kind） | ⚠️ 语义变更 | 旧 event 是带 `kind` 的时间轴表。新版 `event` 是**行为真相**（append-only）；coffee chat / interview 改为 `interaction` / `interview` 强类型表（见 [ADR-0010](./adr/0010-event-fact-dual-truth.md)） |
