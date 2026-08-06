---
type: adr
status: accepted
---

# ADR-0001 · 用 PostgreSQL 而非 SQLite

- 状态：accepted
- 日期：2026-08-01

## 背景（Context）

旧的 jobhunter 原型用 Prisma + SQLite。新后端要重新选型时，用户明确表态：**"后端使用 postgresql，不要使用 sqlite"**。

除了这条明确要求，技术上也确实存在压力：

- 数据模型大量使用 **`jsonb`**（`note.structured`、`position.requirements`、`interview.questions`、`ingest_run.raw_response`）和 **数组列**（`company.aliases`、`person.roles`）。SQLite 只能把它们当 TEXT 存，无法索引、无法在 SQL 里查询。
- 用户要求"任何实体都能当 filter"，这需要 **GIN 索引**、**全文检索 tsvector**、**trigram 模糊匹配公司名/人名**。SQLite 的 FTS5 能做部分，但和 jsonb 组合查询就无能为力。
- 提醒调度器每分钟扫表，同时 ingest 可能在写入长录音的分块结果。SQLite 的**写锁是库级的**，容易出现 `database is locked`。
- 未来要多设备访问（手机小程序 + 电脑网页），SQLite 的单文件形态迟早要换。

## 决策（Decision）

**使用 PostgreSQL 16**，通过 `docker compose` 在本地一键启动，连接串走 `DATABASE_URL`。

ORM 用 SQLModel（SQLAlchemy 2.0 内核），迁移用 Alembic。

## 理由与代价（Consequences）

**得到**

- `jsonb` + GIN 索引：结构化字段可直接进 WHERE 条件
- `text[]` 数组列：别名/角色天然多值
- `tsvector` 生成列 + `pg_trgm`：全文检索与模糊人名/公司名匹配
- MVCC：调度器读与 ingest 写互不阻塞
- 上云零迁移成本（换个 `DATABASE_URL` 即可）

**代价**

- 本地开发需要 Docker（不再是"一个文件拷走就行"）
- 备份从"复制 .db 文件"变成 `pg_dump`
- 首次搭建多一步 `docker compose up -d`

代价可接受：`docker-compose.yml` 已放进仓库，`make dev` 一条命令解决。

## 备选方案（Alternatives）

| 方案 | 为何不选 |
|---|---|
| **SQLite** | 用户明确否决；且 jsonb/数组/GIN/并发写全都缺失，网状查询会退化成应用层内存过滤 |
| **SQLite 起步、后期迁 Postgres** | 迁移时要重写全部 jsonb 字段与索引策略，等于返工。既然终点确定，不如直接到位 |
| **MongoDB** | 文档模型对"通用边 + 强外键混合"的网络图反而更笨拙，且失去事务与引用完整性 |
| **Neo4j（图数据库）** | 网状查询确实更自然，但引入全新技术栈、生态与备份运维成本高。Postgres 的递归 CTE 在 depth ≤ 3、节点 ≤ 200 的个人数据量级完全够用 |
