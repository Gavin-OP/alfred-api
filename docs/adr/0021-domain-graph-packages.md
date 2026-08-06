---
type: adr
status: accepted
---

# ADR-0021 · 领域扩展：共享基座 + 领域 Graph 包

- 状态：accepted
- 日期：2026-08-05
- 关联：[0016](./0016-domain-agnostic-kernel.md)（领域无关内核）、[0017](./0017-organization-generalization.md)（组织泛化）、[0018](./0018-engine-terminology-node-graph-action.md)、[GRAPH.md](../GRAPH.md)。

## 背景（Context）

系统从「求职助理」扩展为「个人 OS」，用户明确的新领域包括：

1. **习惯 Tracker**：弹吉他、打乒乓球等习惯打卡。
2. **作品管理**：书、电影、读后感/观后感。
3. **进度管理**：vlog 进度、项目进度、旅行（今天去哪/明天去哪，与 `location` 挂钩）。

前端形态也从「小程序」演进到「网页 → APP → 树莓派等硬件」。这要求数据模型既能**复用跨领域基元**（人/地点/事件/记忆），又不丢失关系查询能力。

候选数据模型路线：

- **A. 每领域独立关系表**：job/habit/media/progress 各建各的表。查询最强，但新表爆炸。
- **B. 通用 EAV 模型**：一套通用 store，领域只是类型。表最少，但牺牲关系查询——**直接违背 v2「凡需跨内部查询/联结的必须关系化」硬原则**。
- **C. 共享基座 + 领域扩展表**：抽出跨域基元，各 domain 在基座上建扩展表。

## 决策（Decision）

采用 **C 路线：共享基座（Shared Substrate / Kernel）+ 领域扩展表（Domain Packages）**。

### 共享基座（不属于任何 Domain Graph，由跨域 subgraph 维护）
沿用并扩展 ADR-0016 的内核：`person` / `organization` / `occupation` / `skill` / `location`(+`location_alias`) / `event` / `fact` / `chat_message`(+`chat_thread`) / `memory` / `attachment` / `reminder`(+`nudge`/`nudge_rule`)。

> 这些是「个人 OS 的通用词汇」，任意 Domain Graph 可调用，不归某个领域独有。

### 领域 Graph 包（每个 Domain Graph 私有的扩展表）
第一个领域 **job** 现有表**原样保留**，整体包成 Domain Graph：

- **job（求职，已落地）**：`job_posting` `application` `interview` `offer` `experience` `experience_source` `position_requirement` `user_skill` `document_version` `document_version_fact`。
- **habit（习惯，先定模型逐步实现）**：`habit`（定义）+ `habit_log`（打卡，可挂 `location_id`）。
- **media（作品/书影音）**：`work_item`（书/影/剧/游戏，可挂 `location_id`）+ `review`（评分 + 读后感/观后感）。
- **progress（进度/项目/旅行）**：`project`（`kind` 区分 vlog/project/**trip**，支持父子）+ `progress_log` + `trip_day`（旅行按天，挂 `location_id`）。

### 关于旅行
**travel 归入 progress 领域**（`project.kind='trip'`），靠共享 `location` 实现「今天去哪/明天去哪」——不另设独立 travel 领域，避免与 progress 的「按天推进」语义重复。

### 多领域 Graph 的编排
多个 Domain Graph 作为可复用模块，经 **Subgraph** 或外层 **Outer Router** 再编排成总流（见 [ADR-0019](./0019-graph-as-python.md)、[0020](./0020-four-layer-dispatch.md)）。

## 理由与代价（Consequences）

- **得到**：复用跨域基元（`person`/`location`/`event`/`memory`/`attachment`），新领域只增量建少量扩展表，不丢关系查询能力（遵守 v2 关系化原则）。
- **得到**：job 现有表零破坏性迁移——直接包成第一个 Domain Graph，新领域渐进接入。
- **得到**：旅行/习惯场所/作品地点统一挂共享 `location`，「地点」成为跨域连线。
- **代价**：需要为 habit/media/progress 各自设计扩展表与 Node/Action（本期先定模型，逐步实现）。
- **代价**：跨域关联查询（如「某习惯打卡发生在哪个 location、那天还看了什么电影」）需要跨 Graph 的共享 state 协调。

## 备选方案（Alternatives）

- **A. 每域独立表（未选）**：查询最强但新表过多，且与内核已有 `person`/`location` 等共享实体重复建表。
- **B. 通用 EAV（否决）**：牺牲关系查询，直接违反 v2 关系化硬原则；「我的技能 vs 岗位要求匹配」这类 JOIN 查询在 EAV 下极难且慢。
- **job 现有表重写以迎合新模型（未选）**：不必要且高风险；包成 Domain Graph 即可复用。
