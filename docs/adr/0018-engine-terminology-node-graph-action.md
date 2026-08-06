---
type: adr
status: accepted
---

# ADR-0018 · 引擎术语重构：Skill → Node / Graph / Action

- 状态：accepted
- 日期：2026-08-05
- 关联：旧 ADR（0001–0017）中出现的 "Skill" 一律按本 ADR 重新解释为本文件的 **Node**；与 [0012](./0012-orchestration-langgraph.md)（LangGraph 编排）协同。

## 背景（Context）

重构前，"Skill" 一词在本项目里有**两个不同的含义**，长期混用：

1. **引擎处理单元**：一个可带脚本的目录（prompt + output_schema + actions），是编排的基本单元。见 [0004](./0004-scriptable-skill-directories.md)。
2. **领域数据 —— 简历能力词表**：数据模型里的 `skill` 表，以及 `user_skill`（我的能力）、`position_requirement(kind=skill)`（JD 所需）。

同一个词指两件事，导致文档、代码注释、LLM prompt 里反复出现「这个 skill 是引擎的还是简历的？」的歧义；在把系统从「求职助理」扩展为「个人 OS」时，这种混淆会被进一步放大（习惯/作品/进度等每个领域都会有自己的「能力/技能」数据）。

## 决策（Decision）

把**引擎单元**从 "Skill" 改名为三层清晰的概念：

| 新术语 | 含义 | 对应旧物 |
|---|---|---|
| **Action** | 引擎最底层原子写原语，1:1 映射 `services/executor.py` 落库函数 | 不变（原 `ACTION_KINDS`）|
| **Node** | 处理单元 = LLM 抽取（prompt + output_schema）→ 产出 `out` → 跑一组 Action 改共享 state | 原 "Skill" 目录 |
| **Graph** | 带共享 state 的 LangGraph `StateGraph`；多个 Node 经 Edge 串联/分支/并行/循环 | 新增（原 "Skill" 之间没有真控制流）|

明确约定：

- **"skill"（小写，领域数据）** 今后只指 `skill` 能力词表 / `user_skill` / `position_requirement`，不再与引擎混淆。
- **Node** 是引擎处理单元；多个 Node 经 Edge 组成 **Graph**。
- **Action** 是 Node 内部的原子写原语，概念不变。
- 旧 ADR（0001–0017）正文里出现的 "Skill" 一律按 **Node** 理解；"Skills 目录" 方案本身（[0004](./0004-scriptable-skill-directories.md)）仍然有效，只是目录名应从 `skills/` 迁移到 `nodes/`。

## 理由与代价（Consequences）

- **得到**：消除一词双义；`skill` 表恢复其「能力词表」本义，prompt 与文档不再需要反复澄清。
- **得到**：为领域扩展扫清命名空间——每个新领域（habit/media/progress）都能有自己的「能力」数据而不与引擎冲突。
- **代价**：需要一次代码层重命名（skill.yaml → node.yaml，目录 `skills/` → `nodes/`，`ACTION`-相关引用不变）。本轮**只更新文档**，代码重构在后续实现期进行（见 [GRAPH.md](../GRAPH.md) 与 [ADR-0019](./0019-graph-as-python.md)）。
- **代价**：历史 ADR 里的 "Skill" 需读者按本 ADR 重新解释；已在 [GLOSSARY.md](../GLOSSARY.md) 顶部与 [adr/README.md](./README.md) 标注。

## 备选方案（Alternatives）

- **加前缀区分（EngineSkill / DomainSkill）**：过于啰嗦，且 LLM prompt 仍易混。
- **保留 Skill 作为引擎名，用 "capability"/"competency" 指简历能力**：与求职领域已有 `skill` 表冲突，且破坏现有数据模型术语。
- **用户曾提出用 "workflow" 作引擎名**：被否决——`workflow` 只是 Graph 的一种形态，且 LangGraph 的实际抽象是 Graph（带共享 state、可环），"workflow" 不足以表达分支/并行/循环与 subgraph 编排。最终采用 **Node / Graph / Action** 三层。
