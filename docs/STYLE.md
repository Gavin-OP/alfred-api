---
type: contributing
status: 现行有效
---

# 文档规范（STYLE）

> 本文件定义本站点每类文档的统一格式。所有文档（ADR / 设计文档 / CHANGELOG / 协作文档）都应遵循对应格式，保证站点一致、可检索、可维护。
> 修改本规范本身请走 `/domain-modeling` 技能确认。

---

## 1. ADR（架构决策记录）

格式权威定义见 [`adr/README.md`](./adr/README.md)。要点：

- **元信息块**（文件开头）：`状态` / `日期（YYYY-MM-DD）` / `关联`。
- **正文四章节**：背景（Context）→ 决策（Decision）→ 理由与代价（Consequences）→ 备选方案（Alternatives）。
- 已采纳的 ADR 不修改、不删除；改主意写新 ADR 取代（状态标 `superseded by ADR-xxxx`）。
- 现有 22 篇（ADR-0001 ~ ADR-0021）已全部符合此格式。

## 2. 设计文档（Design Doc）

适用：`VISION` / `DATA_MODEL_V2` / `GLOSSARY` / `GRAPH` / `UPLOAD_WORKFLOW` / `design-sessions/*`。

- **标题**：`# 文档名`（可带项目前缀）。
- **元信息块**（紧跟标题后的引用块，统一格式）：

  > **元信息** ｜ 状态：现行有效 ｜ 最后更新：YYYY-MM-DD ｜ 范围：<一句话> ｜ 关联：<ADR/文档链接>

- **正文**：以 `## 1.`、`## 2.` 编号小节组织；关键术语对齐 GLOSSARY。

## 3. CHANGELOG（变更记录）

- 倒序排列：`## YYYY-MM-DD · 标题`。
- 每条含：**核心结论表**（主题 / 决策 / 关联 ADR）+ **关键不变量**（写代码前必须记住的约束）。
- 每条决策链接到对应 ADR 与设计会话存档。

## 4. 协作文档（Contributing）

适用（归入「协作与贡献」分区的文档）：`agents/` 下的领域文档 / Issue 追踪 / 分诊标签（本规范 STYLE 自身的格式见上方各节，不另套用途块）。

- **标题**：`# 文档名`。
- **用途块**（紧跟标题后的引用块）：

  > **用途**：<一句话说明本文档给谁看、解决什么问题>

- **正文**：以清单 / 表格 / 步骤组织，命令与示例用代码块。

---

## 5. 语言与命名

- 全站正文使用**中文**；代码、命令、标识符保留英文。
- 导航分区统一中文命名；文件名保留英文（与 `gh` / 路径兼容）。
- **「最新更新」仅作精选快捷入口**：其中每个文件也必须在各自的规范分区（设计文档 / 架构决策 ADR / 协作与贡献 / 产品需求 PRD）中存在，避免同一内容两处不同步。
- **分类以 YAML frontmatter 的 `type` 字段为准**（见 §6）：`prd` / `design` / `adr` / `vision` / `reference` / `contributing` / `index`。

---

## 6. 治理与变更控制（Governance & Change Control）

本站点所有文档更新都归入以下三类核心类型之一（决议见 [ADR-0022](./adr/0022-documentation-taxonomy-prd-design-adr.md)）：

| `type` | 中文 | 关注点 | 主要读者 | 落地位置 |
|---|---|---|---|---|
| `prd` | 产品需求文档 | **做什么**：功能、应用场景、业务需求、成功指标 | 产品经理 | `docs/prd/` |
| `design` | 设计文档 | **怎么做**：ER 图、数据流图（DFD）、接口、权衡 | 架构师 / 工程师 | `docs/` 设计文档分区 |
| `adr` | 架构决策记录 | **为什么这样决策**：讨论过程、结论、取代了哪些旧 ADR | 全员（尤其未来协作者 / AI） | `docs/adr/` |

其余类型：`vision`（北极星，最高层方向，独立于 PRD）、`reference`（术语表 / 变更记录等参考）、`contributing`（协作与贡献）、`index`（分区索引页）。

### 6.1 三类是"决策与需求"的生命周期链

三者不是互相独立的三个桶，而是由不同决策者拥有、紧密关联的链条：

- **PRD → Design → ADR**：一份 PRD 催生若干 Design Document；Design 在讨论中敲定的关键决定写成 ADR。
- **链接规则**：Design 在 frontmatter `related` 引用它服务的 PRD；ADR 在 `related` 引用它所属的 Design（及被它取代的旧 ADR）。
- `VISION` 是北极星，为 PRD 提供方向，但不等于某个 PRD。

### 6.2 ADR supersede（取代）规则

- 已采纳的 ADR **不修改、不删除**；改主意时写一份新 ADR 取代它。
- 旧 ADR 在 frontmatter 标 `superseded_by: ./adr/00xx.md`，并在正文顶部加醒目标记（如 `> ⚠️ 本 ADR 已被 ADR-00xx 取代`）。
- 新 ADR 在 `related` 里反向引用被取代者。

### 6.3 元信息：YAML frontmatter 为权威来源

每篇文档开头用 YAML frontmatter 声明类型与元信息（机器可读，供 AI 过滤）：

```yaml
---
type: prd | design | adr | vision | reference | contributing | index
status: 现行有效 | 已归档 | proposed | accepted | superseded
updated: YYYY-MM-DD
scope: <一句话说明本文档范围>
related:
  - ./adr/0022-documentation-taxonomy-prd-design-adr.md
---
```

原有的「元信息」引用块仍保留作为人类可读摘要，但 **`type` 以 frontmatter 为准**。

### 6.4 AI 硬规则：用户更新 = 规范（specification）更改

> **任何来自用户的「新功能 / 新意见 / 新决定」，都属于对 specification（规范）的更改，即 design 的更改。**
> AI 在读取或做任何修改前，必须先定位并更新对应的规范文档（PRD / Design / ADR）：
> 1. 新需求 / 业务变化 → 先更新或新建 `prd`。
> 2. 设计 / 模型 / 流程变化 → 先更新 `design` 文档。
> 3. 关键决策变化 → 新增 `adr` 并在旧 ADR 标 `superseded_by`。
> 4. 只有规范文档更新后，代码与其它 markdown 才从规范派生，不得凭空改写散落的文档。

这条规则同时在仓库根 `AGENTS.md` 与 `CONTEXT.md` 中声明，作为 AI 的入口约束。

### 6.5 图表归档位置（Diagram Placement）

图表按类型归属 PRD / Design / ADR 的硬映射，以及「一篇图只归一个分区」的约束，已提升为独立决策记录 [ADR-0023](./adr/0023-diagram-placement.md)。本文不再重复表格。
