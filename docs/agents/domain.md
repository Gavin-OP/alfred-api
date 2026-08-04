# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root — the entry point. It points at the authoritative glossary and the ADRs.
- **`docs/GLOSSARY.md`** — the single source of truth for domain vocabulary. Read it for the area you're about to work in; use its terms exactly.
- **`docs/adr/`** — read ADRs that touch the area you're about to work in. `docs/adr/README.md` is the index (ADRs 0001–0017).
- **`docs/VISION.md`** — the top-level product constraint that overrides local decisions.

If any of these files don't exist, **proceed silently**. Don't flag their absence; don't suggest creating them upfront. The `/domain-modeling` skill (reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates them lazily when terms or decisions actually get resolved.

## File structure

Single-context repo (this repo):

```
/
├── CONTEXT.md              ← entry point
├── docs/
│   ├── GLOSSARY.md        ← authoritative vocabulary
│   ├── VISION.md          ← top-level constraint
│   └── adr/               ← architecture decisions (0001–0017)
└── alfred/                ← backend source
```

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in `docs/GLOSSARY.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal — either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding:

> _Contradicts ADR-0007 (naming Alfred) — but worth reopening because…_
