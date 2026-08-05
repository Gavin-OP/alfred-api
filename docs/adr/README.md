# 架构决策记录（ADR）

**ADR = Architecture Decision Record**。每份 ADR 回答一个问题：**我们当初为什么这么决定？**

代码只告诉你"现在是怎样"，ADR 告诉你"为什么不是别的样子"。半年后你（或任何协作者/AI）回来看，不用凭空猜测取舍原因，也不会把已经权衡过的方案再走一遍弯路。

## 格式

每份 ADR 文件开头是统一的元信息块，正文为四个标准章节：

**元信息块（文件开头）**

- `状态`：`proposed` / `accepted` / `superseded by ADR-xxxx`
- `日期`：采纳日期（`YYYY-MM-DD`）
- `关联`：相关的 ADR / 设计文档链接（可选）

**正文四章节**

- **背景（Context）**：当时面临什么问题、有什么约束
- **决策（Decision）**：最终选了什么
- **理由与代价（Consequences）**：得到什么、付出什么
- **备选方案（Alternatives）**：考虑过但没选的，以及为什么

## 状态

`proposed`（提议中）→ `accepted`（已采纳）→ `superseded by ADR-xxxx`（被取代）
**已采纳的 ADR 不修改、不删除**；改主意就写一份新的来取代它，保留决策的历史轨迹。

## 索引

| # | 标题 | 状态 |
|---|---|---|
| [0001](./0001-postgresql-over-sqlite.md) | 用 PostgreSQL 而非 SQLite | accepted |
| [0002](./0002-two-repositories.md) | 前后端拆成两个独立仓库 | accepted |
| [0003](./0003-taro-react-tri-platform.md) | 前端用 Taro + React 一套三端 | accepted |
| [0004](./0004-scriptable-skill-directories.md) | Skills 用"可带脚本的目录"而非硬编码函数 | accepted |（注：其"拒绝 LangGraph"的替代分析已被 [0012](./0012-orchestration-langgraph.md) 推翻，LangGraph 已采纳；技能目录方案本身仍有效）
| [0005](./0005-chunked-pipeline-for-long-input.md) | 长输入用分块流水线，由 Skill 自行决定 | accepted |
| [0006](./0006-defer-latex-pdf-compilation.md) | MVP 延后 LaTeX → PDF 编译 | **superseded** (by [0014](./0014-resume-provenance.md)) |
| [0007](./0007-naming-alfred.md) | 项目命名为 Alfred，移除 claw 前缀 | accepted |
| [0008](./0008-services-as-single-write-path.md) | services 层作为唯一落库入口 | accepted |
| [0009](./0009-naming-job-posting-occupation.md) | 命名：`job_posting`+`occupation`，退役 `position` | accepted |
| [0010](./0010-event-fact-dual-truth.md) | Event(行为真相) 与 Fact(状态真相) 并列互补 | accepted |
| [0011](./0011-multi-tenant-boundary.md) | 私有表带 user_id，共享实体/词表不带 | accepted |
| [0012](./0012-orchestration-langgraph.md) | 编排：LangGraph 编排 + Action 执行，不用 OpenAI SDK | accepted |
| [0013](./0013-context-segments.md) | 上下文：常驻单 Session + AI 自动分段 + 人工干预 | accepted |
| [0014](./0014-resume-provenance.md) | 简历溯源：软引用 + 渲染强制 footnote + 校验阻断无源 | accepted |
| [0015](./0015-location-entity.md) | Location：`location` 实体 + `location_alias` 反查 | accepted |
| [0016](./0016-domain-agnostic-kernel.md) | 领域无关内核 + 领域包边界 | accepted |
| [0017](./0017-organization-generalization.md) | `company` 泛化为 `organization`（kind 枚举） | accepted |
| [0018](./0018-engine-terminology-node-graph-action.md) | 引擎术语重构：Skill → Node / Graph / Action（消除一词双义） | accepted |
| [0019](./0019-graph-as-python.md) | Graph 是代码不是配置（Graph-as-Code，YAML 只做 Node 配置） | accepted |
| [0020](./0020-four-layer-dispatch.md) | 四层分发 L0–L3（模态→声明式预筛→LLM 模糊回路→回流） | accepted |
| [0021](./0021-domain-graph-packages.md) | 领域扩展：共享基座 + 领域 Graph 包（job/habit/media/progress+travel） | accepted |

> **术语注解**：ADR-0018 起，引擎单元改名 **Node**（原 "Skill" 目录）；旧 ADR（0001–0017）正文里的 "Skill" 一律按 **Node** 理解。"skill"（小写，领域数据）仍指 `skill` 能力词表。
