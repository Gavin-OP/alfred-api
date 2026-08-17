---
type: prd
status: 现行有效
updated: YYYY-MM-DD
scope: <一句话说明本文档范围>
related:
  - ../adr/00xx.md
  - ./design/xx.md
---

# PRD：<产品 / 功能名称>

> 本文件为 PRD 模板，模仿 Google / Microsoft 等大厂 PRD 结构，供 AI 与协作者起手使用。
> **小 PRD 可裁剪**：下表为推荐 section 全集；若某 section 与本次范围无关（例如内部工具没有明确的 Target User），可直接删除该 section，不必保留空标题。文档分类与 frontmatter 键名见 `../STYLE.md`。

## 摘要（Executive Summary）
用 2–4 句话回答：我们要解决什么问题、为谁解决、为什么现在做。让一个没看过上下文的人 30 秒内读懂。

## 目标用户（Target User）
- 主要用户：<角色 / 画像 / 典型场景>
- 次要用户（如有）：<角色>
- 非目标用户（明确排除，避免范围蔓延）：<角色>

## 成功指标（Success Metrics）
可量化、可验证的指标。避免「提升体验」这类不可测表述。
- 指标 1：<定义> 目标 <数值 / 方向>
- 指标 2：<定义> 目标 <数值 / 方向>

## 范围（In Scope）
本次明确要交付的内容：
- <能力 / 边界 1>
- <能力 / 边界 2>

## 非范围（Out of Scope）
本次明确**不**做的内容（防止后续被悄悄加回）：
- <排除项 1>
- <排除项 2>

---

### 可选 section（按需启用，小 PRD 可省略）

## 背景与目标
补充摘要未覆盖的来龙去脉、业务动机与约束。

## 用户场景
关键用户旅程 / 用例清单。

## 功能需求
- FR-1：<需求>
- FR-2：<需求>

## 非功能需求
- NFR-1：<性能 / 安全 / 合规等>

## 关联设计文档（Design）
- [设计文档](./design/xx.md)
- [相关 ADR](../adr/00xx.md)

## 里程碑与 Issue 拆解
- Milestone 1：<目标> → Issue #<n>
- Milestone 2：<目标> → Issue #<n>
