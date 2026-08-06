---
type: adr
status: accepted
---

# ADR-0004 · Skills 用"可带脚本的目录"而非硬编码函数

- 状态：accepted
- 日期：2026-08-01

## 背景（Context）

这条决策经历了**三次修正**，过程本身值得记录。

**第一版（我提的，错的）**：把 skills 理解成后端里的一组 Python 模块——`skills/extract_job.py`、`skills/coffee_chat.py`，每个是我写死的函数。

**用户第一次纠正**：

> "skills 我说的是目前 ai 很火的 skills，可能需要我自己把一些重要流程总结成一个 skills？"

关键点是 **"我自己"**。用户要的不是"开发者预置的能力"，而是**他本人能编写、能持续增补的技能**。硬编码函数意味着每加一个流程都要改 Python 代码 + 重新部署，这条路走不通。

**第二版**：改成"一个 skill = 一个 YAML 文件"，纯声明式。

**用户第二次纠正**：

> "skills 除了 md 和 yaml 以外，也可以有 scripts 啊，是完全可以有 scripts 的，AI skills 是可以写不同的 scripts 的"

这条纠正很重要：纯声明式的 YAML 只能表达"一次调用 + 固定动作"，表达不了「长录音先分段、逐段抽取、按人名合并、再更新画像」这类需要真实控制流的流程。技能必须**能写代码**。

## 决策（Decision）

**一个 Skill = 一个目录**，包含：

```
<skill_name>/
├── skill.yaml     # 必需：身份、路由、分块策略、prompt 模板、输出结构、动作列表
├── run.py         # 可选：async def execute(ctx, inp) -> SkillResult
└── ...            # 任意辅助文件（示例语料、子模板、Markdown 说明）
```

- **无 `run.py`** → 框架走通用流水线：渲染 prompt → 调 AI → 按 `output_schema` 校验 → 执行 `actions` 原语
- **有 `run.py`** → 框架调用 `execute(ctx, inp)`，脚本内可任意编排
- 两者对外接口完全一致，`POST /v1/ingest` 不感知差异
- 启动时扫描 `SKILLS_DIR`（可指向仓库外的私人技能目录），支持 `POST /v1/skills/reload` 热加载
- **加技能 = 加目录，不改后端代码**

脚本默认**返回 Action 列表**而非直接写库，理由见 [ADR-0008](./0008-services-as-single-write-path.md)。

## 理由与代价（Consequences）

**得到**

- 用户可自主扩展能力，这是"把重要流程总结成 skill"的直接实现
- 简单任务写 YAML 就够（低门槛），复杂任务能写 Python（高上限）
- 技能可版本化、可分享、可单独测试
- 后端代码规模不随技能数量增长

**代价**

- 框架层复杂度上升：需要清单校验、模板引擎、动态导入、错误隔离
- `run.py` 是**在主进程直接执行的任意代码**，无沙箱
- 声明式 YAML 需要一套"动作原语"词汇表，用户要学

缓解：
- 单个 skill 加载失败**不影响启动**，标记为 `invalid` 并在 `GET /v1/skills` 暴露原因
- 提供 `POST /v1/skills/{name}/dry-run` 调试端点
- 安全边界写入文档：单人本地场景可接受，多用户上云前必须补沙箱（见 [ROADMAP](../archive/v0.1-baseline/ROADMAP.md) v1.0）

## 备选方案（Alternatives）

| 方案 | 为何不选 |
|---|---|
| **硬编码 Python 函数** | 用户明确要"自己写技能"；每加流程都要改代码+部署 |
| **纯 YAML 声明式** | 表达不了多步控制流（长录音分段合并、条件分支），用户明确要求支持 scripts |
| **纯脚本（无 YAML）** | 简单抽取任务也要写 Python 样板，门槛过高；且路由元数据无处安放 |
| **用 LangGraph / pydantic-ai 等 Agent 框架** | 引入重依赖与学习成本；框架的"节点/图"抽象与用户心智里的"技能包"不匹配；且我们需要的编排复杂度很低 |
| **MCP Server 形式的技能** | 过度设计：MCP 适合跨进程/跨工具集成，这里是同进程内的能力扩展 |
