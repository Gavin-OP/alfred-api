# Alfred Docs (alfred-docs)

This repository is the product/design documentation site for **Alfred**, a personal AI job-hunting assistant. It is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages.

- Source lives in `docs/` (Markdown + a few HTML diagrams).
- Site config: `mkdocs.yml`.
- Deploy: pushing to `main` triggers `.github/workflows/deploy.yml`, which builds and publishes to the `gh-pages` branch (enable it under repo Settings → Pages → `gh-pages`).

## Local preview

```bash
pip install mkdocs-material
mkdocs serve
# open http://127.0.0.1:8000
```

## Structure

- `docs/`
  - `index.md` · `VISION.md` · `DATA_MODEL_V2.md` · `GLOSSARY.md` · `CHANGELOG.md` — current authoritative design
  - `design-sessions/` — raw design session transcripts
  - `adr/` — Architecture Decision Records (ADR-0001 ~ 0021)
  - `diagrams/` — HTML visualizations
  - `agents/` — contributor/agent docs (domain docs, issue tracker, triage labels)
  - `STYLE.md` — documentation format standard for the whole site
  - `archive/v0.1-baseline/` — obsolete v0.1 drafts (kept for reference only; do not implement from these)

## Project entry point

`CONTEXT.md` at the repo root is the single context entry point for AI collaborators and humans.
