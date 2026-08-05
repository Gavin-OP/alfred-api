# ADR-0009 · 命名体系：job_posting + occupation，退役 `position`

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §6.1](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

用户的概念清单里：`Jobs` = 具体招聘实例（公司+岗位+地点），`Positions` = 职位大类（后端工程师 / 产品经理 / 心理咨询师）。
但代码现状把这两个概念叫反了：`position` 表存的是"具体招聘实例"（带 `fingerprint = hash(company_id, title, location)`），`occupation` 表存的是"职位大类"。
`position` 一词在用户和开发者口中指的是相反的东西，会一路污染 API 路径、前端字段、prompt 措辞。**现在改 = 一次迁移；上线后改 = 迁移 + 前端 + 历史数据 + 所有 prompt。**

## 决策（Decision）

- **`job_posting`** = 具体招聘实例（某公司在某地点发布的某个 occupation），带唯一 `fingerprint` / UUID。
- **`occupation`** = 职位大类（software engineer / data scientist / psychologist），跨公司可比。
- 现有 `position` 表**重命名为 `job_posting`**；`occupation` 表保留。
- `position` 一词在本项目**退役**，不再用于任何 API / 前端 / prompt。

## 理由与代价（Consequences）

- ✅ 一词一义，彻底消除歧义；与 O*NET / ISCO 等职业分类标准对齐。
- ✅ 现在迁移成本最低（仅一次重命名 + 外键引用调整）。
- ❌ 需要一次性更新所有引用 `position` 的代码、端点、prompt。

## 备选方案（Alternatives）

- **保持 `position` + `occupation`**：零迁移，但语义互反靠 GLOSSARY 强行统一，脆弱易错。
- **`job` + `position`**：贴合用户直觉，但 `position` 表义反转、且 `job` 在后端常指"后台任务"会与未来 job queue 撞名。
