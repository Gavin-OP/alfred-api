---
type: contributing
status: 现行有效
---

# 领域文档维护规范

> **用途**：供工程技能（engineering skills）与 AI 代理在探索代码库时，了解应如何阅读与消费本仓库的领域文档（CONTEXT / GLOSSARY / ADR / VISION），并遵循统一的术语与冲突标记约定。

本规范说明工程技能（engineering skills）在探索代码库时，应如何消费本仓库的领域文档。

## 阅读顺序

开始探索前，先读这些：

- **仓库根目录 `CONTEXT.md`** —— 单一上下文入口，指向权威术语表与 ADR。
- **`docs/GLOSSARY.md`** —— 领域词汇的唯一权威来源。先读你即将动手的领域相关词条，严格使用其中术语。
- **`docs/adr/`** —— 阅读与你即将动手的领域相关的 ADR；`docs/adr/README.md` 是索引（ADR-0001 ~ 0021）。
- **`docs/VISION.md`** —— 凌驾于局部决策之上的最上层产品约束。

如果以上文件不存在，**静默继续**，不要提示缺失，也不要主动建议创建。领域文档由 `/domain-modeling` 技能（经 `/grill-with-docs`、`/improve-codebase-architecture` 触发）在术语或决策真正敲定时惰性生成。

## 目录结构

本项目是单一上下文仓库（single-context repo）：

```
/
├── CONTEXT.md              ← 入口
├── docs/
│   ├── GLOSSARY.md        ← 权威词汇
│   ├── VISION.md          ← 最上层约束
│   └── adr/               ← 架构决策（0001 ~ 0021）
└── alfred/                ← 后端源码
```

## 使用术语表中的词汇

当你输出的内容命名了一个领域概念（issue 标题、重构提案、假设、测试名），请使用 `docs/GLOSSARY.md` 中定义的说法，不要漂移到术语表明确避免的同义词。

如果需要的概念尚未进入术语表，这是一个信号——要么你在发明项目并不使用的语言（请重新考虑），要么确实存在真实缺口（记下来交给 `/domain-modeling`）。

## 标记 ADR 冲突

如果你的输出与某条已有 ADR 相矛盾，请显式指出，而不是静默覆盖：

> _与 ADR-0007（命名 Alfred）相矛盾——但值得重新开启，因为……_
