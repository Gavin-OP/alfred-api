# ADR-0015 · Location：实体 + 别名反查

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §7.3](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

现状 `position.location` 是自由字符串，"HK" / "香港" / "Hong Kong" 是三个不同值，按地点筛选必然漏。

## 决策（Decision）

- 建 **`location`** 实体：`id, slug, name, country, city, district, admin_code, ...`（结构化字段，可对接地图检索）。
- 建 **`location_alias`**：`canonical_location_id, alias_slug, source(agent|manual)`。
  - 香港 / 香港特区 / Hong Kong / HK → 同一 `location` 行，别名入 `location_alias`。
- `job_posting.location_id` 改为 FK 指向 `location`（替代原字符串字段）。
- 与 ADR-0011 一致：`location` / `location_alias` 为**共享表**（不带 user_id）。

## 理由与代价（Consequences）

- ✅ 结构化检索、可地图可视化；与 `skill_alias` 同一套路，维护成本低。
- ❌ 需别名归一逻辑（agent 抽取时查 alias 表）。

## 备选方案（Alternatives）

- **受控枚举字符串**：比实体轻，但跨名/层级关系表达不了。
- **结构化地理地址（经纬度）**：最严谨，但对求职场景过重。
