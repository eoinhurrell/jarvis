---
name: jarvis
description: Overview and router for the Jarvis local knowledge base — a corporate second-brain of plain markdown notes (no database, no daemon; rg-first, with a structured index derived on demand from YAML frontmatter). Use this skill when the user refers to Jarvis or the KB by name and the request spans the whole KB or you're unsure which sub-skill fits — "what's in the KB", "set up Jarvis", "explain how Jarvis works". It owns the KB model (root, taxonomy, schema) and routes focused work to jarvis-search (find), jarvis-index (add/remove/ingest), and jarvis-doctor (audit/fix). Do NOT trigger on generic note-taking ("add a note", "my notes") — those are better served by a general Zettelkasten tool.
license: MIT
---

# Jarvis — Local Knowledge Base

Jarvis is a corporate second-brain of plain markdown notes. No database, no daemon: every query walks the tree live with `rg`, and the structured index (by type/project/team/status/tag/link) is a **derived view computed on demand** from frontmatter. This keeps the KB portable, harness-agnostic, and impossible to corrupt.

## The Jarvis skill family

Jarvis is split into focused, **independently-installable** skills. This `jarvis` skill is the overview and router — it owns the KB model below. Install the ones you need:

| Skill | Does | Triggers |
|-------|------|----------|
| **jarvis-search** | Find notes: full-text + structured queries | "jarvis, find…", "search the KB for…" |
| **jarvis-index** | Add, update, ingest, or remove notes | "add this to the KB", "ingest this PDF", "remove that note" |
| **jarvis-doctor** | Audit and repair the KB | "fix the KB", "find orphan notes", "lint frontmatter" |

Because each skill installs on its own, every skill carries its own copy of the KB contract. **This skill is the canonical source** — keep the others in sync with it.

## KB root resolution

The KB lives, by default, at **`~/.jarvis`** — create it on first use if it doesn't exist. Resolve the KB root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (overrides the default)
2. A `.jarvis` marker file (walk up from CWD — for a project-local KB)
3. **`~/.jarvis`** (the default home; create if missing)
4. If none of the above is suitable, ask the user and suggest exporting `$JARVIS_KB`

All paths are relative to the KB root.

## Core principle: rg-first, index-derived

There is no stored index. Structured queries (by type/project/team/status/tag) are just frontmatter `rg` patterns; full-text queries are body `rg`. The "index" is a derived view computed on demand.

## Taxonomy

Five top-level note **types** (the `type:` field). Layout is a shallow tree; type lives in frontmatter, not (only) in the path, so notes can move without breaking queries.

```
<kb-root>/
├── projects/      type: project    — active deliverables, time-bound
├── org/           type: org        — company-wide: policy, structure, OKRs
├── teams/         type: team       — pillar/team charters, ownership, runbooks
├── reference/     type: reference  — durable knowledge, how-tos, evergreen
├── decisions/     type: decision   — ADR-style; immutable once accepted
├── sources/        originals of ingested files (PDF/Office/etc.), verbatim
└── inbox/         (untyped capture — to be triaged into the above)
```

Cross-cutting concerns are expressed with `tags:` and wikilinks, **not** deeper folders.

## Frontmatter schema (summary)

Every note opens with YAML:

```yaml
---
id: 20260622T143000     # YYYYMMDDThhmmss; stable, never reused; filename-independent
title: Human Readable Title
type: project|org|team|reference|decision
status: draft|active|blocked|done|accepted|superseded|archived
project: project-slug    # optional cross-ref
team: team-slug          # owning pillar/team
owner: name-or-handle
created: 2026-06-22
updated: 2026-06-22
tags: [retrieval, mlops]
---
```

Outbound links are inline `[[id-or-slug]]` wikilinks in the body — there is **no** `links:` array. Non-markdown files (PDF/Office/…) are ingested into searchable markdown that links back to the stored original under `sources/`.

Full schema, per-type required fields, and the ADR lifecycle: `references/taxonomy.md`.

## Routing

- Want to **find** something? → `jarvis-search`
- **Add / update / ingest / remove** a note? → `jarvis-index`
- **Audit / fix / refactor** the KB? → `jarvis-doctor`

If only this `jarvis` skill is installed (none of the three), you can still answer using the model above and plain `rg`, but installing the focused skill gives the tuned workflow.

## Do NOT use for

- Generic note-taking — "add a note", "my notes", "look this up in my notes", "is there anything on…" → use a general note-taking tool.
- Atomic / permanent / literature notes, or building a knowledge graph → use a dedicated Zettelkasten tool.
- Recording an architecture decision inside a *code repo* → use a code-level ADR tool. (Jarvis's `decisions/` type is for *corporate* decisions recorded in the KB.)

## Guardrails

- Decisions (`type: decision`, `status: accepted`) are immutable — supersede with a new ADR linking back, never edit in place.
- Don't create folders beyond the seven top-level folders without asking.
- Confirm before any operation touching >5 files.
