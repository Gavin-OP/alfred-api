---
type: adr
status: accepted
---

# ADR-0014 · 简历溯源：软引用 + 强制 footnote + 校验阻断

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §7.2](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

愿景明令「简历里每句话都要能溯源」「每个结论要能回答凭什么这么说」。但每句硬绑一个源不现实（AI 润色无法逐句死绑）。

## 决策（Decision）

折中方案（非每句硬绑、非完全不溯源）：
1. **结构化 `fact` 是唯一真相源**。
2. 简历 **section 软引用** 每个 fact（多对多 link 表 `document_version_fact`，不强制每句死绑）。
3. **渲染时强制 footnote**：给用户返回的简历，每处内容附来源标注（`[src: fact:xxx]`）。
4. **校验时阻断无源内容**：validator 拦截没有任何 source 的 bullet，不通过。
5. **允许 polishing**：按目标岗位要求做润色，但必须基于 fact（不凭空生成能力）。

## 理由与代价（Consequences）

- ✅ 可解释、可审计，契合愿景；抽取成本可控。
- ❌ 抽取需维护 fact↔bullet 的 link；validator 复杂度上升。

## 备选方案（Alternatives）

- **每句硬绑（被否决）**：成本过高、约束过死，AI 润色无法落地。
- **不强制（被否决）**：违背愿景「可解释」约束。
