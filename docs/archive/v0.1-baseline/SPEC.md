# Alfred — 个人 AI 助理 · 后端总体设计（SPEC）

> 版本：v0.1-draft ｜ 状态：设计已锁定，进入实现
> 相关文档：[数据模型](./DATA_MODEL.md) ｜ [Skills 系统](./SKILLS.md) ｜ [API 契约](./API.md) ｜ [术语表](../../GLOSSARY.md) ｜ [路线图](./ROADMAP.md) ｜ [ADR](../../adr/README.md)

---

## 1. 产品定位

**Alfred** 是一个面向单人使用的 AI 私人助理（命名取自蝙蝠侠的管家 Alfred Pennyworth）。

- **起步场景**：求职季。职位拆解、心得记录、简历/Cover Letter 版本管理、投递与面试追踪、人脉与 coffee chat 整理、到期提醒。
- **长期定位**：求职季结束后**不报废**，继续作为通用的工作 / 社交 / 知识助理运行。

这条"从求职到通用"的要求，直接决定了两个最核心的架构约束：

| 约束 | 含义 |
|---|---|
| **数据模型领域无关** | 底层是「实体 + 通用边」的网络图，求职概念（Position/Application/Interview）只是其中几类节点，不是骨架。删掉它们，助理依然成立。 |
| **能力可扩展而不改代码** | 新流程以 **Skill（技能目录）** 的形式加入，用户自己写 `skill.yaml`（+ 可选 `run.py`），后端零改动加载。 |

### 1.1 交互形态

用户打开是一个**极简对话界面**，可以发文字、语音、链接、截图。所有输入统一走后端 `POST /v1/ingest`，由 Skill 解析成结构化数据落库。用户体感是"跟秘书说话"，后端做的是"把话变成数据库里的行和边"。

---

## 2. 范围

### 2.1 本阶段（MVP）目标

用户已明确：**先把后端 DB 与 API 全套做扎实**，前端只做最小可调用界面。

MVP 交付：

1. PostgreSQL 完整数据模型（含迁移）
2. 多模态 ingest 管道：文本 / 链接 / 语音(STT) / 截图(视觉)
3. Skills 系统（声明式 + 可带脚本），内置 3 个示例技能
4. AI Provider 抽象层（可插拔 + 假实现供测试 + 规则回退）
5. 全实体 REST CRUD + **通用过滤/图查询接口**
6. 提醒引擎（存储 + 调度 + API 暴露）
7. pytest 测试体系（不依赖真实 API key）

### 2.2 非目标（本阶段明确不做）

| 不做 | 原因 / 何时做 |
|---|---|
| 完整前端 UI | MVP 只需最小调用界面，见 [ROADMAP](./ROADMAP.md) v0.5 |
| LaTeX → PDF 编译 | 需装 ~1GB TeX Live，见 [ADR-0006](../../adr/0006-defer-latex-pdf-compilation.md) |
| 鉴权 / 多用户 | 单人本地场景，v1.0 上云时补 |
| 真实推送（微信/邮件） | MVP 只在 API 暴露到期提醒，前端轮询 |
| Skill 脚本沙箱 | 单机跑自己的脚本，多用户时补 |

---

## 3. 技术栈

### 3.1 后端 `alfred-api`

| 层 | 选型 | 理由 |
|---|---|---|
| Web 框架 | **FastAPI** | 用户熟悉；原生 OpenAPI，直接作为前后端契约源 |
| 契约 / 校验 | **Pydantic v2** | 请求/响应/AI 输出/Skill 清单 全部同一套校验 |
| ORM | **SQLModel** | Pydantic 原生 ORM，模型与 schema 同源，减少重复定义 |
| 数据库 | **PostgreSQL 16** | 见 [ADR-0001](../../adr/0001-postgresql-over-sqlite.md)（jsonb / 数组 / 全文检索 / 并发） |
| 迁移 | **Alembic** | 标准方案，配合 SQLModel |
| 环境与依赖 | **uv** | 极快，`pyproject.toml` 单文件管理 |
| 调度 | **APScheduler** | 轻量，进程内跑提醒扫描 |
| 测试 | **pytest + pytest-asyncio + httpx** | 见 §9 |
| 本地基建 | **docker compose** | 一键起 Postgres，零手工配置 |

### 3.2 前端 `alfred-web`（独立仓库）

**Taro + React** 一套代码编译到 微信小程序 / H5 网站 / RN App。
前后端通过 **OpenAPI 自动生成类型**（`openapi-typescript`）对齐，**不再手写共享类型包**（旧仓库的 `@claw/shared` 手写模式已废弃，`claw` 这个脚手架残留前缀全面移除）。

---

## 4. 系统架构

### 4.1 请求主链路

```mermaid
flowchart TD
    A["客户端<br/>文本 / 链接 / 语音 / 截图"] --> B["POST /v1/ingest"]
    B --> N["Normalizer<br/>STT · 视觉描述 · 链接正文抽取"]
    N --> C{"Skill 选择<br/>显式指定 → 路由技能 → 兜底"}
    C -->|带 run.py| D["用户脚本<br/>execute(ctx, inp)"]
    C -->|纯声明| E["通用流水线<br/>prompt 模板 + output_schema"]
    D --> F["Provider 抽象<br/>调用用户提供的 AI API"]
    E --> F
    F --> G["SkillResult<br/>= 一组 Action 原语"]
    G --> H["services 层<br/>唯一落库入口"]
    H --> I[("PostgreSQL<br/>实体表 + entity_link")]
    B -.审计.-> J[("ingest_request<br/>ingest_run")]
    F -.每次调用.-> J
    I --> K["REST CRUD · /v1/query · /v1/graph · /v1/timeline"]
```

### 4.2 分层与接缝（deep module 原则）

每一层对上层暴露**窄接口**，隐藏内部复杂度。跨层只能通过下表的接缝调用：

| 层 | 目录 | 对外接口（窄） | 内部隐藏（宽） |
|---|---|---|---|
| 路由 | `app/router/` | HTTP 端点 | 参数解析、分页、错误映射 |
| 技能 | `app/skills/` | `execute(ctx, inp) -> SkillResult` | prompt 渲染、分块、重试、合并去重 |
| 服务 | `app/services/` | **Action 原语函数**（`upsert_company` …） | upsert 去重、链接建立、事务、审计字段 |
| 供应商 | `app/providers/` | `LLMProvider` / `STTProvider` / `VisionProvider` Protocol | 各家 SDK 差异、退避重试、JSON 修复 |
| 模型 | `app/models/` | SQLModel 表 | 索引、约束、生成列 |

**最关键的一条铁律：**

> `services` 层是**唯一**的落库入口。REST 路由和 Skill **共用同一组 Action 原语**。

这保证 AI 侧和人工侧永远不会出现"两套校验逻辑漂移"。Skill 想建一家公司，走的是和 `POST /v1/companies` 完全相同的函数。

### 4.3 为什么 Skill 返回 Action 而不是直接写库

Skill（尤其带 `run.py` 的用户脚本）**默认返回一个 Action 列表**，由框架统一执行。好处：

- **可审计**：每个 Action 连同其来源 skill / chunk 记入 `ingest_run`
- **可 dry-run**：`POST /v1/ingest?dry_run=true` 只返回将要执行的 Action，不落库
- **可事务**：整批 Action 在一个事务里提交，失败整体回滚
- **可重放**：解析结果保留，改了落库逻辑可重跑

脚本仍保留 `ctx.services` 逃生舱用于复杂场景，但文档明确标注为"绕过审计，慎用"。

---

## 5. 数据模型（摘要）

完整字段、枚举、索引见 **[DATA_MODEL.md](./DATA_MODEL.md)**。这里只讲设计思想。

### 5.1 网络图：节点 + 通用边

```mermaid
erDiagram
    COMPANY ||--o{ POSITION : "发布"
    POSITION ||--o{ APPLICATION : "投递"
    APPLICATION ||--o{ INTERVIEW : "面试轮次"
    APPLICATION ||--o| OFFER : "结果"
    APPLICATION }o--|| DOCUMENT_VERSION : "使用的 CV/CL 版本"
    DOCUMENT_ASSET ||--o{ DOCUMENT_VERSION : "版本历史"
    PERSON }o--o{ INTERVIEW : "面试官"
    ENTITY_LINK }o--o{ NOTE : "任意实体互链"
    ENTITY_LINK }o--o{ EVENT : "任意实体互链"
    ENTITY_LINK }o--o{ PERSON : "任意实体互链"
```

- **强关系**用外键（Company→Position→Application→Interview/Offer），保证引用完整性。
- **弱/任意关系**用通用边表 `entity_link(source_type, source_id, target_type, target_id, relation)`。
  一条笔记可以同时链到"某公司 + 某岗位 + 某个人 + 某类技能"，无需为每种组合建表。

### 5.2 "任何东西都能当 filter"如何实现

用户要求：*每一个职位、每一类职位、每一家公司、每一个人、每一天、某一件事、某一类型的事，都可以成为 filter。*

落地为一个**统一查询端点** `POST /v1/query`，接受组合条件：

| 维度 | 字段 | 支撑结构 |
|---|---|---|
| 某个实体 | `related_to: {type, id}` | `entity_link` 双向索引 + 外键 |
| 某一类职位 | `tags: ["job_family:quant"]` | `tag` + `entity_tag` |
| 某家公司 | `company_id` | 外键（含 Application 上的冗余列） |
| 某个人 | `person_id` | `entity_link` |
| 某一天 / 时间段 | `date_from` / `date_to` | 各表 `occurred_at` / `event` 时间轴 |
| 某类事 | `event_kind` | `event.kind` |
| 全文 | `q` | `tsvector` 生成列 + GIN |

返回统一的 `EntityRef` 列表或时间轴，前端可直接渲染。

### 5.3 求职者真实痛点在模型中的体现

以下场景在设计阶段已确认要覆盖，均已落到具体字段（详见 DATA_MODEL）：

| 场景 | 落点 |
|---|---|
| 已读不回 / ghosting | `application.status = ghosted`，`position.status = watching\|abandoned` |
| 多轮面试链路 | `interview.round_no` + `interview.kind`（OA/笔试/电面/技术/HR/终面） |
| Offer 横向对比与谈判 | `offer` 表薪资拆项 + `comparison_note_md` + `negotiation_log_md` |
| Recruiter / 内推人关系网 | `person.roles[]` + `application.referrer_person_id / recruiter_person_id` |
| 签证 / 地点约束（HK / JobsDB 场景） | `position.visa_sponsorship` / `work_mode` / `location` |
| 投递指标看板 | `GET /v1/stats/funnel` 按 `application.status` 聚合 |
| 黑名单避雷 | `company.is_blacklisted` / `person.is_blacklisted` + 原因 |
| 求职季后延续 | `note` / `event` / `person` / `tag` 全部领域无关，可独立使用 |

### 5.4 可观测性表（robust 的关键）

`ingest_request` + `ingest_run` 记录每一次输入、每一次 AI 调用（含 chunk 序号、模型、token、耗时、原始响应、解析结果、错误）。
这让"AI 解析"从黑盒变成**可重试、可审计、可算成本、可 debug** 的过程。

---

## 6. Skills 系统（摘要）

完整规范见 **[SKILLS.md](./SKILLS.md)**。

- 一个 Skill = **一个目录**：`skill.yaml`（必需）+ `run.py`（可选）+ 任意辅助文件。
- **无脚本**：走通用流水线（渲染 prompt → 调 AI → 按 `output_schema` 校验 → 执行 `actions` 原语）。
- **有脚本**：框架调用 `execute(ctx, inp) -> SkillResult`，脚本内可自由编排（多次调 AI、写 Python 逻辑、调 services）。
- 两者**对外接口完全一致**，`/v1/ingest` 不感知差异。
- **加技能 = 加目录，不改后端代码。** 支持 `POST /v1/skills/reload` 热加载。

### 6.1 长输入：单步 还是 分块？（用户明确提问的点）

**答案：由 Skill 自己按输入长度决定，对外接口不变。**

- 短输入（默认 ≤ 6000 字符）→ **单步**：一次调用返回完整结构，最快最稳。
- 长输入（10 分钟 coffee chat 录音、长文章）→ **分块流水线**：

```mermaid
flowchart LR
    A[原始输入] --> B[STT / 视觉 / 正文抽取]
    B --> C{超长?}
    C -->|否| D[单步调用]
    C -->|是| E[按话题或字符切段<br/>带重叠]
    E --> F[逐段调用 AI<br/>同一 output_schema]
    F --> G[合并去重<br/>按 dedupe key]
    G --> H[可选 reduce 调用<br/>生成全局摘要]
    D --> I[Action 列表]
    H --> I
```

分块的价值：避开上下文上限；**任何一段失败可单独重试**，不会像"一个巨型 JSON"那样一处格式错就全崩。

---

## 7. AI Provider 抽象

### 7.1 接口（Protocol）

```python
class LLMProvider(Protocol):
    async def complete_json(self, *, system: str, user: str,
                            schema: dict, model: str | None = None) -> LLMResult: ...
    async def complete_text(self, *, system: str, user: str,
                            model: str | None = None) -> LLMResult: ...

class VisionProvider(Protocol):
    async def describe_image(self, *, image: bytes, mime: str, prompt: str) -> LLMResult: ...

class STTProvider(Protocol):
    async def transcribe(self, *, audio: bytes, mime: str,
                         language: str | None = None) -> Transcript: ...
```

### 7.2 实现与配置

| 实现 | 说明 |
|---|---|
| `OpenAICompatibleProvider` | 覆盖 OpenAI / DeepSeek / Kimi 等一切 OpenAI 兼容端点，靠 `LLM_BASE_URL` 切换 |
| `GeminiProvider` | Google 原生协议 |
| `WhisperSTTProvider` | Groq / OpenAI Whisper |
| `FakeProvider` | 测试专用，读 fixture，**不需要真实 key** |

用户的 API key 通过 `.env` 注入（`LLM_API_KEY` / `STT_API_KEY`），**永不进代码、永不进前端**。

### 7.3 健壮性策略

1. **超时 + 指数退避重试**（`tenacity`，默认 3 次）
2. **JSON 修复**：模型输出带 markdown fence / 尾逗号时自动清洗后再解析
3. **Schema 校验失败 → 一次纠错重试**（把校验错误回喂给模型）
4. **规则回退**：LLM 完全不可用时，用正则/启发式抽取最低限度信息（公司名、URL、日期），保证输入**永不丢失**——至少落成一条 `note` + 原始 `attachment`
5. `GET /v1/llm-ping` 连通性探测（沿用旧后端已验证的做法）

---

## 8. 提醒引擎

- 表：`reminder`（`due_at` / `recurrence_rule` / `status` / `target_entity`）
- 调度：APScheduler 每 60 秒扫描到期项 → 状态置 `sent` → 写入一条 `event(kind=reminder_fired)`
- MVP 通道：`in_app`。前端通过 `GET /v1/reminders?status=due` 拉取。
- 真实推送（微信 / 邮件）作为 `channel` 的新实现后置，**不改调用方**。
- 来源：用户手动创建，或 Skill 通过 `create_reminder` 原语自动创建（例："下周记得投" → 自动生成提醒）。

---

## 9. 测试策略（TDD）

**红 → 绿 → 重构**，每个 milestone 先写失败测试。

| 层级 | 内容 | 依赖 |
|---|---|---|
| **服务层单测** | Action 原语：upsert 去重、链接建立、事务回滚 | 真实测试库（事务内回滚） |
| **Skill 单测** | 清单解析、prompt 渲染、分块切分、结果合并去重、schema 校验失败处理 | **FakeProvider**，零网络 |
| **API 契约测** | httpx AsyncClient 打全部端点，校验 OpenAPI 响应结构 | 依赖注入覆盖 provider |
| **回归夹具** | 真实场景语料（JD 截图文本、coffee chat 转写）作为 fixture | 离线 |

铁律：**任何测试都不得依赖真实 API key 或外网**。CI 里 `pytest` 必须能在无 key 环境跑通。

---

## 10. 配置

`.env`（参考 `.env.example`，真实值永不提交）：

```
DATABASE_URL=postgresql+psycopg://alfred:alfred@localhost:5432/alfred
LLM_PROVIDER=openai_compatible
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_API_KEY=***
LLM_MODEL=deepseek-chat
VISION_MODEL=
STT_PROVIDER=whisper
STT_API_KEY=***
STT_MODEL=whisper-large-v3
SKILLS_DIR=app/skills
ATTACHMENT_DIR=./var/attachments
TIMEZONE=Asia/Hong_Kong
LOG_LEVEL=INFO
```

---

## 11. 仓库落地

| 仓库 | 路径 | 内容 |
|---|---|---|
| `alfred-api` | `../alfred-api`（与 `jobhunter` 同级） | 本文档所述后端 |
| `alfred-web` | `../alfred-web` | Taro + React 三端前端（v0.5 起） |
| `jobhunter`（旧） | 保留只读 | Node/Express/Prisma/SQLite 原型，**不再演进**，可作为规则回退逻辑的参考 |

两仓库独立 git / CI / 部署，理由见 [ADR-0002](../../adr/0002-two-repositories.md)。

---

## 12. 待决问题（Open Questions）

| # | 问题 | 影响 | 计划 |
|---|---|---|---|
| Q1 | 附件存本地磁盘还是对象存储 | 上云时迁移成本 | MVP 本地 + 抽象 `Storage` 接口，v1.0 再切 S3 兼容 |
| Q2 | 截图 OCR 用视觉大模型还是 OCR 引擎 | 成本 vs 精度 | 先视觉大模型（省依赖），量大再加本地 OCR |
| Q3 | LaTeX 编译放宿主机还是独立容器 | 部署复杂度 | v0.3 评估，倾向独立容器（隔离 + 不污染主镜像） |
| Q4 | Skill 路由是否需要每次调 LLM | 成本 | 先关键词预筛 + LLM 兜底；统计后再优化 |
