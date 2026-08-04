# ADR-0020 · 四层分发（4-Layer Dispatch, L0–L3）

- 状态：accepted
- 日期：2026-08-04
- 关联：[0018](./0018-engine-terminology-node-graph-action.md)、[UPLOAD_WORKFLOW.md](../UPLOAD_WORKFLOW.md)、[GRAPH.md](../GRAPH.md)。

## 背景（Context）

当用户输入进来，外层 **Outer Router**（见 [ADR-0019](./0019-graph-as-python.md)）需要把它分发到正确的 Domain Graph。难点在于**模糊/口语输入**——例如：「昨天跟 Sarah 去看了《奥本海默》，顺便练了会儿吉他」同时触达 media（电影）、habit（吉他）与共享的 person（Sarah）。个人管家的体感，正取决于这种复合输入能否被正确处理。

候选分发策略：

- 纯声明式打分（`registry.select` 关键词/URL/模态）：快、确定，但口语模糊输入易误判。
- 纯 LLM 分类器：最灵活，但每次多一次 LLM 调用、且非确定。
- 混合：声明式预筛做快路径，模糊输入回落 LLM。

## 决策（Decision）

采用**四层分发**，逐层升级代价，最后一层做回流：

| 层 | 名称 | 做法 | 触发条件 |
|---|---|---|---|
| **L0** | 模态归一化 | 把 text / link / image / audio 归一为统一的 `SkillInput` | 每次输入入口 |
| **L1** | 声明式预筛 | `registry.select()` 关键词/URL/模态打分，命中即走对应 Domain Graph | 有明确信号（如招聘 URL）|
| **L2** | LLM 模糊回路 | 仅当 L1 命中**模糊/复合/多实体**时才调一次 LLM 做意图分类 + 领域路由 | L1 无法唯一确定 |
| **L3** | 回流 | 用户纠错/补信息后带上下文回到对应 Graph 重跑 | 用户修正或补充 |

关键约定：

- **L1 → L2 的闸门**：文本输入先经 L1 预筛；只有当实体命中处于模糊状态（多领域冲突、无高置信匹配）才进 L2，避免每个输入都烧一次 LLM。
- **跨域 subgraph 常驻**：`memory` / `capture` / 聊天不作为任何 Domain Graph 私有，而是常驻 subgraph，任意 Domain Graph 都能调用。复合输入（看电影+见人）的处理不是「选一个图」，而是「进入编排好的跨域 subgraph」。
- **L3 回流是闭环而非失败**：用户纠偏是正常路径，回流带原始上下文回到对应 Node/Graph 重跑。

## 理由与代价（Consequences）

- **得到**：个人管家场景下的复合口语输入被正确拆解（电影→media Graph，吉他→habit Graph，Sarah→共享 person subgraph）。
- **得到**：快路径（L1）零 LLM 调用、确定性强；模糊路径（L2）灵活但有闸门控制成本。
- **得到**：与 ADR-0012「LangGraph 之上极薄 runtime」一致——分发本身就是一个 LangGraph 外层编排。
- **代价**：L2 引入一次额外 LLM 调用（仅模糊输入）；需要维护 `registry.select` 的打分规则与 LLM 分类器的 prompt。
- **代价**：L3 回流要求 Graph 状态可恢复（LangGraph `interrupt`/`resume` 支撑）。

## 备选方案（Alternatives）

- **纯声明式（淘汰 LLM）**：对口语复合输入几乎必然误判，损害管家体感。
- **纯 LLM（每次都分类）**：非确定性强、每次输入成本翻倍、且对明确信号（招聘 URL）浪费算力。
- **Graph 只是领域命名空间（用户已否决）**：见 [0018](./0018-engine-terminology-node-graph-action.md)——Graph 是真控制流，多领域 Graph 经外层 router / subgraph 编排，不是命名空间。
