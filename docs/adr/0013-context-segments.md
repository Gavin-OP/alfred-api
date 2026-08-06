---
type: adr
status: accepted
---

# ADR-0013 · 上下文管理：常驻单 Session + 自动分段

- 状态：accepted
- 日期：2026-08-04
- 关联：[design-sessions/2026-08-03-v1-redesign.md §7.1](../design-sessions/2026-08-03-v1-redesign.md)

## 背景（Context）

聊天是连续的（类微信），需要定期压缩，否则 token 爆炸。但三种极端都不行：
- 严格 Session 隔离 → 太割裂，不适合连续聊天产品；
- 纯手动开新话题 → 太累；
- 无限常驻不切 → token 爆炸。

## 决策（Decision）

- **常驻单 `chat_thread`**（每用户一个），不做频繁切换。
- thread 内切成语义 **`chat_segment`**：AI 检测主题漂移自动开新段；用户可手动重命名 / 合并 / 归档。
- **上下文组装 = 混合模式**：每次请求拼 `(相关 segment 的摘要) + (当前 segment 最近 N 轮) + (向量/BM25 检索到的 fact/event)`。
- 这是「带分段的时间线」。

## 理由与代价（Consequences）

- ✅ 原话永久留存（呼应愿景）、token 可控、用户可干预分段。
- ✅ 压缩/摘要按 segment 粒度进行，不丢历史。
- ❌ 主题漂移检测可能误判（仅做"建议开启新会话"，不打断用户）。
- ❌ 需一个 context-assembly 服务做检索 + 组装。

## 备选方案（Alternatives）

- **AI 自动拆主题（静默）**：体验丝滑但误判难调试、用户不知上下文边界。
- **严格 Session 隔离（OpenAI 模式）**：最干净但最割裂，不适合连续聊天。
