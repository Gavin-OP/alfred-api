---
type: adr
status: accepted
---

# ADR-0016 · 领域无关内核 + 领域包边界

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §7.5](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

愿景：Alfred 从求职工具演变为个人管家（健康、记账、社交）。若 `event` / `fact` 现在就带求职专有字段，V3 要重来。内核必须领域无关。

## 决策（Decision）

- **领域无关内核（通用，V3 直接复用）**：`event`(行为) / `fact`(状态) / `person` / `organization` / `occupation` / `skill` / `skill_alias` / `location` / `location_alias` / `chat_thread` / `chat_segment` / `chat_message` / `memory` / `reminder` / `nudge` / `nudge_rule`。
- **求职领域包（V1 专属，未来可平行加健康包/记账包）**：`job_posting` / `application` / `interview` / `experience` / `experience_source` / `user_skill` / `document_version` / `document_version_experience` / `document_version_fact` / `resume_asset`。
- **原则**：内核表不带求职专有字段；新增领域时不改内核，只加领域包表，并通过 `fact.kind` / `event.type` 的枚举扩展接入。

## 理由与代价（Consequences）

- ✅ 保证 V3「同一内核 + 领域包」的演进路径，跨域联动（如"面试撞体检"）成为可能。
- ❌ 初期要谨慎区分内核/领域边界，避免把求职字段漏进内核。

## 备选方案（Alternatives）

- **领域无关全 EAV（fact 三元组）**：极致灵活但所有查询自连接，性能/可读性痛，且与强类型表并存会冗余。
- **单库加求职表**：V3 时表爆炸、语义混杂。
