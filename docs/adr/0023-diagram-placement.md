---
type: adr
status: accepted
---

# ADR-0023 · 图表归档位置（Diagram Placement）

- 状态：accepted
- 日期：2026-08-06
- 关联：[STYLE §6](../STYLE.md)、[PRD 求职助理](../prd/job-seeking.md)、[ADR-0022 文档治理](../0022-documentation-taxonomy-prd-design-adr.md)

## 背景（Context）

文档治理（ADR-0022）确立了 PRD / Design / ADR 三类文档。实践中反复出现一类分歧：某张图（如 User Flow、Sequence Diagram、Decision Tree）究竟该放进哪类文档。不统一会导致同一张图在多处重复、或找不到归属。需要在治理框架下给出明确的图表 → 文档映射。

## 决策（Decision）

图表按类型固定归属，且**一篇图只归一个分区**：

| 文档类型 | 归档的图表 |
|---|---|
| **PRD** | User Flow、UX Flow（以用户视角为主）；Business Flow、Process Flow |
| **Design Document** | System Context Diagram、Sequence Diagram、Module Diagram、ER Diagram / Schema Diagram、State Machine、Petri Net（如有） |
| **ADR** | Decision Tree、Cutoff Diagram |

补充约束：跨边界的图只在一处表达。例如 User Flow 中的 "AI 解析" 步骤，PRD 只描述用户视角，其内部时序另出 Sequence Diagram 落 Design Document，不在 PRD 重复。

## 理由与代价（Consequences）

- ✅ 消除「图该放哪」的反复争议，PRD / Design / ADR 各自边界清晰。
- ✅ 一篇图单归属，避免重复维护与漂移。
- ❌ 某些图天然跨界（如 State Machine 既像设计也像决策），个别情况需人工判断归类。

## 备选方案（Alternatives）

- **不规定归属，由作者自由放置**：会导致重复与失联，已否决。
- **全部图都放 Design Document**：违背 PRD 以用户视角为主的定位，已否决。
- **用标签（tag）而非固定分区**：增加认知负担，不如硬映射直接，已否决。
