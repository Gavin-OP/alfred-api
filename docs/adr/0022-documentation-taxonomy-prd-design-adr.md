---
type: adr
status: accepted
---

# ADR-0022 · 文档治理：PRD / Design / ADR 三类分类

- 状态：accepted
- 日期：2026-08-06
- 关联：[STYLE §6](../STYLE.md)、[GLOSSARY](../GLOSSARY.md)

## 背景（Context）

站点文档此前只有「设计文档」与「ADR」两类隐式分区，没有 PRD，且"需求与设计"是一个混装分区（既含 VISION、又含 GLOSSARY / CHANGELOG）。用户要求所有文档更新明确归入三类：

1. **PRD（产品需求文档）**：关注"做什么"——功能、应用场景、业务需求。
2. **Design Document（设计文档）**：关注"怎么做"——ER 图、数据流图（DFD）、接口、权衡，供架构师 / 工程师。
3. **ADR（架构决策记录）**：关注"为什么这样决策"——讨论过程、结论、取代哪些旧 ADR，类似会议纪要。

并要求：每次用户的新功能 / 新意见都属于 specification（规范）更改，即 design 更改；AI 必须基于规范文档做改动。

## 决策（Decision）

- 文档站采用三类核心类型，外加 `vision`（北极星）、`reference`（术语表 / 变更记录）、`contributing`（协作与贡献）、`index`（分区索引）。
- 三者是**决策与需求的生命周期链**：PRD → Design → ADR，彼此用 frontmatter `related` 链接；VISION 为 PRD 提供方向但不等于 PRD。
- 每篇文档以 **YAML frontmatter 的 `type` 字段**为权威分类（机器可读，供 AI 过滤），同时保留「元信息」引用块作人类摘要。
- **ADR supersede 规则**：已采纳 ADR 不修改、不删除；被取代时旧 ADR 标 `superseded_by` 并在正文顶部加 `⚠️ 已被 ADR-00xx 取代` 标记。
- **AI 硬规则（spec-driven）**：任何用户更新先落到对应规范文档（PRD / Design / ADR），再从规范派生代码与其它 markdown，不得改写散落文档。该规则同时写入根 `AGENTS.md` 与 `CONTEXT.md`。

## 理由与代价（Consequences）

- 三件套对齐大厂通行实践（PRD / 设计文档 / ADR），且通过链接避免三套文档互相漂移。
- frontmatter `type` 让 AI 能按类型检索与过滤，落实"基于规范做改动"。
- 代价：每篇文档需维护 frontmatter；新增 PRD 需先有 PRD 分区。可接受。

## 备选方案（Alternatives）

- *三分类互不相干*：rejected——易漂移。
- *ADR 并入 Design 子章节*：rejected——失去独立决策序列与会议纪要式可追溯性。
- *不引入 frontmatter，仅用导航分区体现类型*：rejected——AI 无法机器过滤。
