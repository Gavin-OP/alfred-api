---
type: index
status: 现行有效
---

# 最近更新

> 本页是站点的「最新更新」快捷索引。下列文件均可在各自规范分区（架构决策 ADR / 需求与设计 / 协作与贡献）找到完整正文，此处仅作入口汇总。

## 2026-08-06

- [求职助理用户流程 PRD](./prd/job-seeking.md) — 工作流一至十五（JD 采集 → 投递 → 面试 → Offer/拒信 → 比较 / 保温 / 统计 / Coffee Chat / 合同审查）+ FR-1～FR-22；修复流程图 mermaid 语法并补齐缺失图
- [数据模型 V3（求职领域细化 · 草稿）](./DATA_MODEL_V3.md) — follow_up.kind / activity_log / contract_review / person·interaction 扩展
- [产品需求 PRD 分区](./prd/README.md) — 登记求职助理用户流程 PRD

## 2026-08-05

- [ADR-0021 领域图包](./adr/0021-domain-graph-packages.md) — 共享基座 + 领域 Graph 包（job / habit / media / progress+travel）
- [ADR-0020 四层分发](./adr/0020-four-layer-dispatch.md) — L0–L3：模态 → 声明式预筛 → LLM 模糊回路 → 回流
- [ADR-0019 图即 Python](./adr/0019-graph-as-python.md) — Graph 用代码声明而非配置
- [ADR-0018 引擎术语 Node / Graph / Action](./adr/0018-engine-terminology-node-graph-action.md) — 消除 "Skill" 一词双义

## 2026-08-04

- [数据模型 V2](./DATA_MODEL_V2.md) — V1 重新设计定稿（`job_posting` / `occupation` / `event`+`fact` …）
- [引擎规范 GRAPH](./GRAPH.md) — Node / Graph / Action 三层抽象权威规范

## 2026-08-03

- [重新设计会话 2026-08-03](./design-sessions/2026-08-03-v1-redesign.md) — V1 重新设计完整讨论存档
