---
type: index
status: 现行有效
updated: 2026-08-06
scope: 产品需求（PRD）分区落地页与编写指引
related:
  - ../VISION.md
  - ../STYLE.md
---

# 产品需求 PRD

> **用途**：本分区存放**产品需求文档（PRD）**——关注"我们要做什么"：具体功能、应用场景、业务需求与成功指标。面向产品经理，也为架构师 / 工程师提供设计输入。

## 与本站其它类型的关系

| 类型 | 关注点 | 关系 |
|---|---|---|
| **VISION（北极星）** | 最高层方向 | PRD 的方向来源，但 VISION ≠ 某个 PRD |
| **PRD（本分区）** | 做什么 | 催生 Design Document |
| **Design Document** | 怎么做（ER / DFD / 接口） | 由 PRD 派生，讨论中产出 ADR |
| **ADR** | 为什么这样决策 | 记录 Design 中的关键决定，可取代旧 ADR |

分类规则见 [文档规范 STYLE §6](../STYLE.md)。

## 如何新增一篇 PRD

1. **从模板起手**：复制 [`TEMPLATE.md`](./TEMPLATE.md)，其章节已对齐本说明。
2. 在 `docs/prd/` 下新建 `NNN-<slug>.md`，开头用 YAML frontmatter 声明 `type: prd`。
3. 必备章节（**小 PRD 可删除无关 section**，不必保留空标题）：
   - 摘要（Executive Summary）
   - 目标用户（Target User）
   - 成功指标（Success Metrics）
   - 范围（In Scope）
   - 非范围（Out of Scope）
   - 关联 Design
4. 可选章节（按需启用）：背景与目标 / 用户场景 / 功能需求 / 非功能需求 / 里程碑与 Issue 拆解。
5. 在 `mkdocs.yml` 的「产品需求 PRD」分区登记。
6. 相应设计落 `design` 文档，关键决策落 `adr`。

## 现有 PRD

- [求职助理用户流程](./job-seeking.md) — V1 首个产品路径：工作流一至十五（JD 采集 → 投递 → 面试 → Offer/拒信 → 比较 / 保温 / 统计 / Coffee Chat / 合同审查）+ FR-1～FR-22；关联设计文档 [DATA_MODEL_V3](../DATA_MODEL_V3.md)。
