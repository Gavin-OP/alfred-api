# CONTEXT.md — Alfred Project Entry Point

> This is the project's **single context entry point**. Any AI collaborator or human should read this first, then drill down as needed:
> - **Glossary (authoritative)**: [docs/GLOSSARY.md](docs/GLOSSARY.md)
> - **Decision records**: [docs/adr/](docs/adr/) (with README index)
> - **Vision & evolution**: [docs/VISION.md](docs/VISION.md)
> - **Data model**: [docs/DATA_MODEL_V2.md](docs/DATA_MODEL_V2.md)
> - **Upload / capture workflow**: [docs/UPLOAD_WORKFLOW.md](docs/UPLOAD_WORKFLOW.md)
> - **Engine spec (Node/Graph/Action)**: [docs/GRAPH.md](docs/GRAPH.md)
>
> Domain-doc maintenance rules live in `docs/agents/domain.md` (driven by the `/domain-modeling` skill).

---

## 1. Purpose

Alfred is a **domain-agnostic "personal OS / personal butler"**. It starts as a job-hunting assistant; the kernel stays domain-agnostic and is extended through **Domain Graphs** to cover: jobs (job), habits (habit), media/books (media), and progress/projects/travel (progress). All domains share the same engine, the same shared substrate, and the same memory & context.

## 2. Kernel / Domain boundary

- **Shared substrate (Kernel)**: `person` / `organization` / `occupation` / `skill` / `location` / `event` / `fact` / `chat_*` / `memory` / `attachment` / `reminder` / `nudge`. Belongs to no single domain; maintained by cross-domain subgraphs. Boundary and rationale: [ADR-0016](docs/adr/0016-domain-agnostic-kernel.md).
- **Domain Package**: each domain is a **Domain Graph** with its own private extension tables; only the private tables carry `user_id`. The first domain, job, is live; habit / media / progress are added this round (model first, implement later).

## 3. Engine: Node / Graph / Action (three-layer abstraction)

> **Terminology refactor (2026-08-04)**: the old word "Skill" meant both an engine unit and a capability vocabulary, causing conflicts. The engine unit is now renamed **Node**; multiple Nodes form a **Graph**; **Action** is the atomic write primitive inside a Node. "Skill" in old ADRs = today's Node. See [ADR-0018](docs/adr/0018-engine-terminology-node-graph-action.md).

| Layer | Name | Role | Form |
|---|---|---|---|
| Bottom | **Action** | Atomic write primitive, 1:1 mapped to a `services` function, single write entry | `services/executor.py` + `ACTION_KINDS` |
| Middle | **Node** | Processing unit: LLM extraction (prompt + output_schema) → produces `out` → runs a set of Actions to mutate shared state. **No control flow.** | `alfred/nodes/<name>/node.yaml` (config) |
| Top | **Graph** | LangGraph `StateGraph` with shared state; Nodes chained/branched/parallel/looped via Edges. Domain Graphs can be wrapped as subgraphs and re-orchestrated by an outer router | `alfred/graphs/<domain>.py` (**Python, not YAML**) |

- **Declarative Node**: `node.yaml`-driven (`match` / `chunking` / `prompt` / `output_schema`).
- **Scripted Node**: Python-body-driven, may return `Command(goto=...)` for **data-driven dynamic routing** (static flow still drawn in the Graph).
- **Routing (4-layer dispatch L0–L3)**: modality → declarative pre-filter → LLM fuzzy loop → fallback. Cross-domain `memory`/`capture` subgraphs are always resident and callable from any Domain Graph. See [UPLOAD_WORKFLOW.md](docs/UPLOAD_WORKFLOW.md).
- **dry_run / commit**: preview and confirmation before an Action writes; migrated to LangGraph `interrupt` / `resume` (HITL) so dirty data never lands.

## 4. Data Model

Normalize first: anything that needs cross-internal querying / joining must be relational, not a JSON array. (Iron Law #6 in [DATA_MODEL_V2.md](docs/DATA_MODEL_V2.md))

- Unique keys: UUID primary key; `slug` (company) / `fingerprint` (role = sha256(company_id + normalized title + location)) for business de-duplication.
- Skill three-meaning split + equal-weight storage: `skill` is only a shared vocabulary; "my skills" = `user_skill`, "JD requires" = `position_requirement(kind=skill)`; similar skills reverse-looked-up via `skill_alias`.
- Experience store: one row per experience, storing only the raw spoken draft `raw_input_md`; AI-polished bullets go into `document_version_fact` per role — no separate bullets table.

## 5. Memory & Context

- `chat_message` (full two-party conversation, timestamped) is the base — every word you say is retained and retrievable later.
- `chat_thread` groups/archive + `memory` entities (extracted structured facts/preferences/intents/context) for long-term accumulation.
- Recall = BM25 full-text + vector cosine hybrid weighting (7:3) + exponential time-decay salience; PG uses `<=>` + tsvector, SQLite degrades to Python-side cosine + LIKE (wrapped in `MemoryStore`, exposing only `recall()` to callers).

## 6. Vocabulary summary (must stay consistent)

| Term | Meaning | Easy to confuse with |
|---|---|---|
| **Node** | Engine processing unit (formerly "Skill") | not `skill` (capability vocabulary) |
| **Graph** | LangGraph StateGraph, Nodes joined by Edges | not a namespace |
| **Action** | Atomic write primitive inside a Node | — |
| **Domain Graph** | A domain's Node+Edge set (reusable module) | — |
| **skill** | Capability vocabulary (domain data) | not the engine Node |
| **Kernel / Shared substrate** | Cross-domain tables (person/organization/skill/location/...) | belongs to no Domain Graph |
| **L0–L3** | 4-layer dispatch: modality / pre-filter / LLM fuzzy / fallback | — |

## 7. Dev Workflow (Vibe Coding)

Natural-language requirement → `/grilling` (relentless interview to finalize, grill-with-docs emits ADR/glossary/CHANGELOG in sync) → `to-spec` pushes the spec to the GitHub issue tracker (labeled `ready-for-agent`) → Agent implements to spec → TDD verification.

Engineering skills (Matt Pocock collection) are configured under `docs/agents/`; issue tracking and triage labels are in `docs/agents/issue-tracker.md` and `docs/agents/triage-labels.md`.

## 8. Documentation change control (spec-driven)

Every user update (new feature / opinion / decision) is a **specification change** — i.e. a design change. AI must update the canonical spec doc **first**:

- New requirement → `prd` (see `docs/prd/`).
- Design / model / flow change → `design` doc.
- Key decision change → new `adr`, mark old `superseded_by`.

Code and other markdown derive from the spec, never the reverse. Taxonomy: `docs/STYLE.md` §6; decision: [ADR-0022](docs/adr/0022-documentation-taxonomy-prd-design-adr.md). Each doc declares its type in YAML frontmatter (`type`: `prd` / `design` / `adr` / `vision` / `reference` / `contributing` / `index`).
