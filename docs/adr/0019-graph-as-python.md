# ADR-0019 · Graph 是代码，不是配置（Graph-as-Code）

- 状态：accepted
- 日期：2026-08-05
- 关联：[0012](./0012-orchestration-langgraph.md)（LangGraph 编排方向）、[0018](./0018-engine-terminology-node-graph-action.md)（术语）、[GRAPH.md](../GRAPH.md)（规范）。

## 背景（Context）

确定 Node / Graph / Action 三层后，下一个分叉点是：**「Node 之间怎么连（控制流）」这件事写在哪？**

候选：

- 纯声明式 `graph.yaml`：每个 Domain Graph 一个 yaml，列出 nodes + edges + 条件路由。
- 纯 Python 代码直接建 `StateGraph`。
- 混合：yaml 声明 + 允许 Node 返回动态 `Command(goto=...)`。

LangGraph 的 `StateGraph` 本身就是用 Python API 编排 Node/Edge/条件路由的——它从来就不是为「声明式配置驱动」设计的。把控制流塞进 YAML 既丢失类型安全，也表达不了复杂循环与数据驱动的路由。

## 决策（Decision）

**Graph = Python 代码（Graph-as-Code）。** 具体约定：

1. **Graph 用 Python 声明**：每个 Domain Graph 是 `alfred/graphs/<domain>.py`，用 LangGraph `StateGraph` API 画 Node/Edge/条件路由（支持 DAG 与环）。作者是写 Python 的人。
2. **YAML 只做 Node 配置**：每个 Node 的 `node.yaml`（原 `skill.yaml`）只描述身份（name/domain/version）、路由（match）、分块（chunking）、prompt、output_schema。**YAML 绝不承载控制流**（不画 edge、不写条件分支）。
3. **Node 内部只在数据驱动的动态路由时返回 `Command(goto=...)`**：静态流程一律在 Graph 代码里画好；当路由必须根据运行期数据（如「某字段命中才跳分支」）才在 Node 内返回 `Command` 交回 runtime，作为逃生舱，不作为常态画法。

### 形态对照

```
Graph（Python, alfred/graphs/<domain>.py）:
    builder = StateGraph(AlfredState)
    builder.add_node("extract_job", extract_job_node)
    builder.add_edge("extract_job", "commit")
    builder.add_conditional_edges("route", {job: ..., habit: ...})
        ↓ 编译
    langgraph StateGraph（带共享 state，可串/分/并/环）

Node（配置, alfred/nodes/<name>/node.yaml）:
    name: extract_job
    domain: job
    match: { keywords: [...], priority: 80 }
    prompt: ...
    output_schema: ...
    # 没有 edges / 没有控制流
```

## 理由与代价（Consequences）

- **得到**：控制流有完整类型安全与 IDE 支持；复杂循环、并行、条件分支表达自然；与 LangGraph 设计意图一致（ADR-0012「LangGraph 之上极薄 runtime」）。
- **得到**：配置（YAML）与代码（Python）职责清晰分离——Node 的「是什么、怎么抽」与 Graph 的「怎么连」解耦。
- **得到**：Vibe Coding 友好——作者改流程就改 Python（明确、可测），改 Node 行为就改 YAML（无需动控制流）。
- **代价**：重组一个流需要改 Python 而非只改 YAML（用户已明确接受这一点）。
- **代价**：Graph 代码需要单元测试覆盖 Edge 拓扑（在 [codebase-design](../../docs/agents/) 的「小接口深实现」约束下做）。

## 备选方案（Alternatives）

- **纯声明式 `graph.yaml`（被否决）**：用户明确判断「YAML 只适合描述有哪些 Node、属于哪个 domain、怎么写配置；不适合做控制流」。分支/并行/循环/动态路由在 YAML 里既难读又弱类型。
- **纯 Python 但 Node 也硬编码进 Graph**：保留 `node.yaml` 配置更好——Node 的 prompt/output_schema 是高频改动点，应数据化、不进代码。
- **混合（Node 内常态返回 Command 画静态流）**：被否决——静态流程在 Graph 里画一次即可，Node 内只在真正数据驱动时返回 `Command`，避免控制流散落在各处难以追踪。
