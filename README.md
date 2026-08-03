# Alfred 文档站 (alfred-docs)

本仓库是 **Alfred 个人 AI 求职助理** 的产品设计文档站，由 [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 构建并部署到 GitHub Pages。

- 源文件在 `docs/` 目录（Markdown + 少量 HTML 可视化）。
- 站点配置：`mkdocs.yml`。
- 部署：推送到 `main` 即由 `.github/workflows/deploy.yml` 自动构建并发布到 `gh-pages` 分支（仓库 Settings → Pages 选择 `gh-pages` 分支即可）。

## 本地预览

```bash
pip install mkdocs-material
mkdocs serve
# 打开 http://127.0.0.1:8000
```

## 目录

- `docs/`
  - `index.md` · `VISION.md` · `DATA_MODEL_V2.md` · `GLOSSARY.md` · `CHANGELOG.md` — 当前权威设计
  - `design-sessions/` — 设计会话原始记录
  - `adr/` — 架构决策记录（ADR-0001 ~ 0017）
  - `diagrams/` — HTML 可视化
  - `archive/v0.1-baseline/` — 已作废的旧版 v0.1 草稿（仅供追溯，请勿照此实现）
