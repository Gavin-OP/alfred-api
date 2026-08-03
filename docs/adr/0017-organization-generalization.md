# ADR-0017 · Organization 泛化（company → organization）

- 状态：accepted
- 关联讨论：[design-sessions/2026-08-03-v1-redesign.md §8.1](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

现有 `company` 装不下学校、社团、NGO、政府机构、基金、开源办公室 —— 而 Coffee Chat 的对象、经历的所属组织经常不是公司。V2 社交 CRM 更需要泛化。

## 决策（Decision）

- `company` 表**重命名为 `organization`**，新增 `kind` 枚举：`company` / `school` / `club` / `ngo` / `government` / `fund` / `open_source_office` / `other`。
- 覆盖：港大/中大（school）、红十字会/入境处（ngo/government）、社团（club）等。
- 引用它的外键（job_posting、experience、person.employer 等）同步改指向 `organization`。
- 与 ADR-0011 一致：`organization` 为**共享表**（不带 user_id）。

## 理由与代价（Consequences）

- ✅ V2 社交 CRM 直接受益；一次迁移成本可控（一张表 + 若干外键）。
- ❌ 需重写所有引用 `company` 的代码与迁移。

## 备选方案（Alternatives）

- **本轮不动，V2 再泛化**：遇到非公司实体只能塞错表，后面迁移更痛。
- **新建 organization，company 兼容保留**：引入冗余，查询要 UNION。
