---
type: design
status: 现行有效
---

# UPLOAD_WORKFLOW — 捕获与分发工作流

> **元信息**
>
> - 状态：现行有效
> - 最后更新：2026-08-05
> - 范围：用户输入从进入到落库的完整链路
> - 关联：[GRAPH](./GRAPH.md)、[ADR-0020](./adr/0020-four-layer-dispatch.md)

> 配套 [GRAPH.md](./GRAPH.md) 与 [ADR-0020](./adr/0020-four-layer-dispatch.md)。
> 描述一次用户输入从进入到落库的完整链路：**4 层分发（L0–L3）** + **Mode A 捕获家族** + **Mode B 组装/查询**。对应产品侧工作流见 [求职助理用户流程 PRD](./prd/job-seeking.md)（工作流一至十五）。

---

## 1. 总览

```
用户输入 ──▶ [L0 模态] ──▶ [L1 声明式预筛] ──▶ [L2 LLM 模糊回路] ──▶ 选中 Domain Graph
                                                                      │
        ┌─────────────────────────────────────────────────────────────┘
        ▼
  Domain Graph（Node 经 Edge 串联）
   ├─ 跨域 subgraph：memory / capture / chat（常驻，任意图可调用）
   └─ 捕获类 Node 跑完 → pending_actions（dry_run 预览）→ interrupt
        │
        ▼
  [L3 回流] 客户端渲染 ui_intent → 用户确认/纠错 → resume → ActionExecutor 落库
```

---

## 2. 4 层分发（L0–L3）

个人管家场景的输入高度口语化、跨领域、常复合（"昨天跟 Sarah 看了《奥本海默》顺便练了会儿吉他"）。分发必须**快路径确定性 + 模糊路径智能**。

| 层 | 名称 | 做什么 | 触发条件 |
|---|---|---|---|
| **L0** | 模态归一化 | text / link / image / audio → 统一 `SkillInput`；音频转写、图片 OCR/VLM 抽取 | 所有输入入口 |
| **L1** | 声明式预筛 | `registry.select()` 按关键词 / URL / 模态 / priority 打分，高置信直接命中 Domain Graph | 信号明确（如招聘 URL → job 图）|
| **L2** | LLM 模糊回路 | 仅当 L1 实体命中**模糊/复合**时，进一次 LLM 分类器，判定走哪个 Domain Graph + 意图 | L1 置信度低或跨域复合 |
| **L3** | 回流 | 用户纠错 / 补信息后，回到对应 Graph 重跑对应 Node | 确认阶段用户修改 |

> **为什么混合**：纯声明式（L1 only）对口语模糊输入太弱；纯 LLM（L2 only）每次多一次调用且非确定。混合 = L1 快路径 + L2 仅模糊回落，契合 [ADR-0012](./adr/0012-orchestration-langgraph.md)「LangGraph 之上极薄 runtime」。

### L2 分类器输出示例
```json
{ "domain": "media", "intent": "add_work", "entities": { "title": "奥本海默", "kind": "movie" },
  "also": [ {"domain":"habit","intent":"log_habit","entities":{"name":"吉他"}} ] }
```
复合输入可被拆成多个领域意图，分别进入各自 Domain Graph（共享 `person`/`location` 基座自然去重）。

---

## 3. Mode A：捕获家族（由分发选择 Node）

| 领域 | Node | 落点 |
|---|---|---|
| job | `extract_job` `capture_experience` `record_application` `record_interaction` `generate_resume` `interview_log` | 求职包各表 |
| habit | `log_habit`（打卡，可挂 `location_id`）| `habit` / `habit_log` |
| media | `add_work`（书/影/剧/游戏，可挂 `location_id`）`write_review` | `work_item` / `review` |
| progress | `log_progress`（vlog/项目）`plan_trip` `log_trip_day`（按天挂 `location_id`）| `project` / `progress_log` / `trip_day` |
| 跨域 | `capture_memory`（兜底，自由内容→`memory`）`capture_note` | `memory` / `note` |

> 自由内容无明确领域匹配时，回退 `capture_memory`（替代原 `capture_note` 兜底）。

---

## 4. Mode B：组装 / 查询（基于已有数据）

不进入捕获流，而是基于已落库数据组装或查询：

- **改简历 / 生成文档**：`generate_resume` 读 `document_version` + `document_version_fact`，套 `resume_asset` 模板编译。
- **技能匹配查询**（三大类）：① 哪些岗位需要我的技能 X；② 我凭技能 X 能投哪些岗位；③ 我的技能 vs 岗位要求匹配度（matched/gap）—— 全靠 `user_skill ⋈ skill ⋈ position_requirement` 三表 JOIN。
- **查截止 / 进度 / 旅行足迹**：跨领域查询，靠共享 `location` / `person` 基座 JOIN。
- **主动提醒（Nudge）**：`nudge_rule` 定时扫描生成 `nudge` 实例，与用户手设 `reminder` 分表不混。

---

## 5. 跨域 Subgraph（常驻）

`memory` / `capture` / `chat` 三个 subgraph 不属于任何 Domain Graph，由外层 router 或任意图 `add_subgraph` 复用：

- **chat**：把双方对话写入 `chat_message` / `chat_thread`（记忆底座）。
- **capture**：多模态附件落 `attachment`，语音/截图经 `experience_source` / `application_evidence` 挂接。
- **memory**：从输入/对话抽取结构化 `memory`（事实/偏好/意图/上下文），供后续召回。

---

## 6. 关键不变量

1. **脏数据不入库**：捕获类 Node 只推 `pending_actions`，经 `interrupt`/`resume` 确认后才由 `ActionExecutor` 落库。
2. **Action 唯一写入口**：Graph/Node 不直接写 DB。
3. **共享基座去重**：跨领域实体（person/location/organization/skill）走共享表，不重复建。
4. **无头引擎**：Graph 只产出 `state` + `ui_intent`，客户端形态（小程序/网页/APP/硬件）不影响引擎。
