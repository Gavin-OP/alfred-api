# Alfred REST API 契约（v1 草案）

> ⚠️ **本文件属于 v0.1 基线草稿，已作废。** 当前权威规范见顶层 [`DATA_MODEL_V2`](../../DATA_MODEL_V2.md)、[`GLOSSARY`](../../GLOSSARY.md)、[`ADR 0009–0017`](../../adr/README.md)。本文档仅供历史追溯。
>
> 配套：[SPEC](./SPEC.md) ｜ [数据模型](./DATA_MODEL.md) ｜ [Skills](./SKILLS.md)
> 运行后可在 `/docs`（Swagger）与 `/openapi.json` 看到自动生成的权威契约。本文件是设计意图说明。

---

## 0. 通用约定

| 项 | 约定 |
|---|---|
| 前缀 | `/v1` |
| 成功响应 | **直接返回资源对象或列表**（不再包 `{ok, data, error}` 信封——FastAPI 原生风格，OpenAPI 更干净） |
| 错误响应 | `{"detail": {"code": "...", "message": "...", "hint": "..."}}` + 恰当 HTTP 状态码 |
| 列表分页 | `?limit=50&cursor=<opaque>` → `{"items": [...], "next_cursor": "..." }` |
| 时间 | 一律 ISO-8601 带时区（UTC 存储，`TIMEZONE` 仅影响展示与调度） |
| ID | UUID 字符串 |
| 软删除 | `DELETE` 默认归档（`archived_at`），`?hard=true` 才物理删 |
| 鉴权 | MVP 无（单人本地）。预留 `X-Alfred-Token` 头，开关由 `AUTH_ENABLED` 控制 |

> **注意**：旧 jobhunter 后端的 `{ok, data, error}` 信封**不再沿用**。`alfred-web` 是新前端，直接按 OpenAPI 生成类型。

---

## 1. 系统

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/health` | 存活 + 数据库连通 |
| `GET` | `/v1/llm-ping` | **AI 供应商连通性探测**：返回 provider/model/延迟/是否可用 |
| `GET` | `/v1/meta/vocab` | 受控词表：relation 列表、状态枚举、job_family、tag namespace（前端下拉直接用） |

---

## 2. Ingest —— 一切输入的入口

### `POST /v1/ingest`

接受 `application/json` 或 `multipart/form-data`（带文件时）。

```jsonc
{
  "text": "今天看到字节的量化研究员岗，8月30号截止，我挺想去的，下周记得投",
  "link": "https://www.linkedin.com/jobs/view/123456",
  "attachment_ids": ["<uuid>"],          // 先传附件再引用
  "skill": "extract_job",                 // 可选，显式指定技能；不填则自动路由
  "hints": "这是朋友转发给我的",            // 可选，附加上下文
  "source_kind": "wechat_mp",             // 可选
  "dry_run": false                        // true 只返回将执行的 Action，不落库
}
```

**响应 `200`**

```jsonc
{
  "request_id": "<uuid>",
  "status": "succeeded",                  // received|running|partial|succeeded|failed
  "skill": "extract_job",
  "skill_version": 1,
  "summary_md": "已记录 **字节跳动 · 量化研究员**，截止 8/30，并在 8/27 提醒你投递。",
  "created": [
    {"type": "company",  "id": "...", "label": "字节跳动", "is_new": false},
    {"type": "position", "id": "...", "label": "量化研究员", "is_new": true},
    {"type": "reminder", "id": "...", "label": "下周投递"}
  ],
  "warnings": ["未能识别薪资范围"],
  "runs": [{"stage": "extract", "chunk_index": 0, "model": "deepseek-chat",
            "latency_ms": 2310, "status": "ok"}]
}
```

**要点**

- **输入永不丢**：即使 AI 全挂，也会落一条 `note` + 附件，`status=partial` 并在 `warnings` 说明。
- 长输入自动走分块（由 skill 决定），`runs` 会有多条。
- 大文件/长音频建议 `async=true`（v0.2 加）→ 立即返回 `request_id`，客户端轮询。

### 其他

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/v1/ingest/{id}` | 查询处理状态与全部 `runs`（调试/成本核算） |
| `POST` | `/v1/ingest/{id}/retry` | 重跑，可指定 `chunk_index` **只重试失败的那一段** |
| `GET` | `/v1/ingest?status=failed` | 失败列表 |

---

## 3. 附件

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/v1/attachments` | `multipart` 上传截图/录音/PDF。按 sha256 去重。返回 `{id, kind, sha256}` |
| `POST` | `/v1/attachments/{id}/transcribe` | 显式触发 STT / 视觉描述（ingest 时会自动做） |
| `GET` | `/v1/attachments/{id}` | 元数据 + 转写文本 |
| `GET` | `/v1/attachments/{id}/raw` | 原文件 |

---

## 4. 实体 CRUD

统一模式（`{entity}` ∈ `companies` / `positions` / `people` / `applications` / `interviews` / `offers` / `notes` / `events` / `reminders` / `tags` / `documents`）：

| 方法 | 路径 |
|---|---|
| `GET` | `/v1/{entity}` — 列表 + 过滤 + 分页 |
| `POST` | `/v1/{entity}` — 创建（走**与 Skill 相同的 Action 原语**） |
| `GET` | `/v1/{entity}/{id}` — 详情（`?expand=links,tags,timeline`） |
| `PATCH` | `/v1/{entity}/{id}` — 局部更新 |
| `DELETE` | `/v1/{entity}/{id}` — 归档 |

### 各实体的专属过滤参数

| 端点 | 过滤 |
|---|---|
| `/v1/positions` | `company_id` `status` `job_family` `seniority` `work_mode` `visa_sponsorship` `deadline_before` `discovered_after` `interest_min` `q` |
| `/v1/applications` | `status` `company_id` `channel` `submitted_after` `followup_before` |
| `/v1/people` | `role` `company_id` `followup_before` `q` |
| `/v1/notes` | `kind` `occurred_from` `occurred_to` `salience_min` `tag` `q` |
| `/v1/events` | `kind` `from` `to` `person_id` |
| `/v1/reminders` | `status`（含 `due` 语义糖：pending 且 `due_at<=now`）`target_type` `before` |
| `/v1/interviews` | `application_id` `outcome` `scheduled_from` `scheduled_to` |

### 嵌套便捷端点

```
GET  /v1/companies/{id}/positions
GET  /v1/positions/{id}/applications
GET  /v1/applications/{id}/interviews
POST /v1/applications/{id}/status      { "status": "interviewing", "note": "..." }   # 状态机校验
POST /v1/interviews/{id}/interviewers  { "person_id": "...", "role": "main" }
```

---

## 5. 简历 / Cover Letter

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET/POST` | `/v1/documents` | 文档档案（`kind=resume\|cover_letter`） |
| `GET` | `/v1/documents/{id}/versions` | 版本链 |
| `POST` | `/v1/documents/{id}/versions` | 新增版本（提交 LaTeX 源码，可带 `based_on_version_id`） |
| `GET` | `/v1/documents/{id}/versions/{no}` | 某版完整源码 |
| `GET` | `/v1/documents/{id}/versions/{a}/diff/{b}` | 两版差异 |
| `POST` | `/v1/documents/{id}/tailor` | **AI 定制**：给定 `position_id`，返回修改建议或直接生成新版本（`apply=true`） |
| `POST` | `/v1/documents/{id}/versions/{no}/compile` | **v0.3+**：编译 PDF。MVP 返回 `501 Not Implemented` |

> 用户核心诉求"忘了当初投递用的哪份 CV" → `GET /v1/applications/{id}?expand=documents` 一次返回当时的 CV/CL 版本号与完整源码。

---

## 6. 关联与图（网状能力）

| 方法 | 路径 | 说明 |
|---|---|---|
| `POST` | `/v1/links` | 建边 `{source:{type,id}, target:{type,id}, relation, note}`。`relation` 必须在受控词表内 |
| `DELETE` | `/v1/links/{id}` | 删边 |
| `GET` | `/v1/entities/{type}/{id}/links` | 该实体的全部边（双向） |
| `GET` | `/v1/entities/{type}/{id}/graph?depth=2` | 子图（节点上限 200，深度上限 3），前端可直接画关系图 |

---

## 7. 通用查询：`POST /v1/query`

**这是"任何东西都能当 filter"的落地端点。**

```jsonc
{
  "types": ["note", "event", "position", "application"],   // 要哪些实体，空=全部
  "related_to": [{"type": "person", "id": "<uuid>"}],       // 和谁/什么有关（走 entity_link 双向）
  "tags": ["job_family:quant", "skill:python"],
  "company_id": "<uuid>",
  "event_kind": ["coffee_chat"],
  "status": ["interviewing"],
  "date_from": "2026-07-01",
  "date_to": "2026-07-31",
  "q": "估值模型",                                            // 全文检索
  "sort": "occurred_at:desc",
  "limit": 50
}
```

**响应**：统一 `EntityRef` 列表 —— `{type, id, label, subtitle, occurred_at, snippet, tags[]}`，前端一套组件即可渲染任何组合。

### `GET /v1/timeline?from=&to=&kinds=`

按天分桶的时间轴（"我这一天/这一周都做了什么"），合并 event / application / interview / note。

### `GET /v1/stats/funnel?from=&to=&group_by=month`

投递漏斗：投递数 / 回复数 / 面试数 / Offer 数 + 回复率、面试率，按月分桶。

### `GET /v1/stats/activity`

活跃度热力图数据（每天产生了多少条记录），给自己正反馈。

---

## 8. Skills 管理

| 方法 | 路径 | 说明 |
|---|---|---|
| `GET` | `/v1/skills` | 全部技能：名称/版本/描述/状态（`ok`\|`invalid`）/加载错误 |
| `GET` | `/v1/skills/{name}` | 清单详情 + 最近运行统计（成功率、平均延迟、token） |
| `POST` | `/v1/skills/reload` | 热重载技能目录 |
| `POST` | `/v1/skills/{name}/dry-run` | 拿一段样例输入试跑，只看产出的 Action，不落库（写 skill 时的调试利器） |

---

## 9. 错误码

| code | HTTP | 含义 |
|---|---|---|
| `validation_error` | 422 | 请求体不合法（Pydantic） |
| `not_found` | 404 | |
| `conflict` | 409 | 唯一约束冲突（slug / fingerprint / sha256） |
| `invalid_status_transition` | 409 | 状态机不允许的跃迁 |
| `unknown_relation` | 422 | relation 不在受控词表 |
| `skill_not_found` | 404 | |
| `skill_invalid` | 500 | 技能清单/脚本有问题 |
| `provider_unavailable` | 503 | AI 供应商不可用（此时 ingest 仍会降级保存原始输入） |
| `schema_violation` | 502 | AI 多次返回不合规结构 |
| `not_implemented` | 501 | 如 MVP 的 PDF 编译 |

---

## 10. 前端类型生成

```bash
# alfred-web 仓库内
npx openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts
```

后端改契约 → 前端重跑一行命令 → TypeScript 立即报错指出不兼容处。**这取代了旧仓库手写 `@claw/shared` 的做法**，从根上杜绝前后端类型漂移。
