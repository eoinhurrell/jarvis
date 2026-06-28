---
name: jarvis-knowledgebase
description: Manage a local corporate knowledge base / second brain of markdown notes. Use this skill when the user explicitly refers to Jarvis or the corporate KB by name — "jarvis, find...", "search the KB for...", "add this to the KB", "ingest this PDF into Jarvis", "where in the KB did we decide X" — to capture, find, organize, write, validate, or refactor notes about their projects, org, pillar/team, reference material, or decisions. Do NOT trigger on generic phrasing like "add a note", "my notes", "look this up in my notes", or "is there anything on..." — those are better served by a general note-taking / Zettelkasten tool. The KB is a flat-ish markdown tree with rich YAML frontmatter; search is rg-driven with a structured index derived on demand (no persisted index). Non-markdown files are ingested into searchable markdown that links back to the stored original.
license: MIT
---

# Jarvis — Local Knowledge Base Manager

Jarvis manages a corporate second-brain of plain markdown notes. No database, no daemon: every query walks the tree live with `rg`, and the "structured index" (by type/project/team/status/tag/link) is a **derived view computed on demand** from frontmatter. This keeps the KB portable, harness-agnostic, and impossible to corrupt.

## When to use

Trigger when the user refers to Jarvis or the corporate KB **by name** — e.g. "jarvis, find…", "search the KB for…", "add this to the KB", "ingest this PDF into Jarvis", "where in the KB did we decide X".

## Do NOT use for

- Generic note-taking — "add a note", "my notes", "look this up in my notes", "is there anything on…" → use a general note-taking tool.
- Atomic / permanent / literature notes, or building a knowledge graph → use a dedicated Zettelkasten tool.
- Finding latent links between existing notes → use a link-suggestion tool.
- Surfacing non-obvious syntheses across a vault → use a synthesis tool.
- Recording an architecture decision inside a *code repo* → use a code-level ADR tool. (Jarvis's `decisions/` type is for *corporate* decisions recorded in the KB.)

If the intent is ambiguous, ask which surface the user means before acting.

## KB root resolution

The KB lives, by default, at **`~/.jarvis`** — create it on first use if it doesn't exist. Resolve the KB root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (overrides the default)
2. A `.jarvis` marker file (walk up from CWD — for a project-local KB)
3. **`~/.jarvis`** (the default home; create if missing)
4. If none of the above is suitable, ask the user and suggest exporting `$JARVIS_KB`

All paths below are relative to the KB root.

## Core principle: rg-first, index-derived

There is no stored index. To answer any structured question, run `rg` over frontmatter and synthesize the answer. Structured queries (by type/project/team/status/tag) are just frontmatter `rg` patterns; full-text queries are body `rg`. `references/search-recipes.md` has tuned patterns for every field, wikilinks, tags, ranked full-text, and the structured rollups (orphans, broken links) — **read it before composing any non-trivial query.**

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

Cross-cutting concerns are expressed with `tags:` and wikilinks, **not** deeper folders. Resist folder proliferation; the graph carries the relationships.

Full schema, required/optional fields per type, and the ADR lifecycle: `references/taxonomy.md`. Read it before creating or refactoring notes.

## Frontmatter schema (summary)

Every note opens with YAML. Common fields:

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

Outbound links are inline `[[id-or-slug]]` wikilinks in the body — there is **no** `links:` array. The link-audit `rg` recipes scan the body for `[[...]]`. Authoritative field rules per type are in `references/taxonomy.md`.

## Capabilities & workflows

### 1. Search / query (most common)
- **Full-text**: `rg` with recipes from `references/search-recipes.md`. Return note title + id + path + matching line, ranked by match density.
- **Structured** ("active projects owned by team X", "accepted decisions tagged retrieval"): compose a frontmatter `rg` query per `references/search-recipes.md` (multi-field queries chain `rg` with `-l` + a second pass, recipes given there).
- Always surface the `id` and path so the user (or you) can open/link it.
- **If a result note has a `source:` block, it was ingested from a non-markdown original.** Read its `source.path` (relative to the KB root) and present that file to the user — via your harness's file-presentation tooling if available — not just the derived markdown. This is how a PDF/xlsx/etc. surfaces from a text search and the user gets the real document back.

### 2. Create / update notes
- Read `references/taxonomy.md` for the type's required fields.
- Generate `id` from current timestamp (use the `date` command, or a time MCP tool if available); filename `<id>-<slug>.md`.
- Place in the type's folder (or `inbox/` if triaging later).
- On update, bump `updated:` and never rewrite `id` or `created`.
- After writing, eyeball frontmatter against the schema in `references/taxonomy.md` (required fields present, valid `status`/`type`).

### 2b. Ingest non-markdown documents (PDF, Word, Excel, PPT, CSV, HTML, images)
Jarvis searches markdown only, so other formats are converted to searchable markdown that links back to the stored original. **Read `references/ingestion.md` before ingesting anything** — it defines the storage model, the `source:` frontmatter contract, and the per-format extraction steps you run directly (CLI tools and Python libraries — no helper script, no sibling skills).
- Procedure (you run this directly): hash-and-dedup first, generate an `id`, copy the original verbatim into `sources/<id>/`, extract faithful text with the format-appropriate tool, and write a derived note with a `source:` block (path + sha256 + kind + ingester).
- Never summarize during ingestion — keep extracted text verbatim so `rg` hits land.
- The derived note behaves like any other note for search; what makes it special is that it resolves back to a real file (see §1).

### 3. Maintenance (link audit, orphans, hygiene)
All derived from `rg` — see the "Maintenance" section of `references/search-recipes.md` for the exact pipelines:
- **Orphans**: notes whose `id` appears in no other note's links and that declare no outbound links.
- **Broken links**: `[[id]]` references whose target `id` doesn't exist.
- **Frontmatter lint**: notes missing required fields or carrying an invalid `type`/`status`.
- Propose fixes; don't bulk-rewrite without confirming.

### 4. Refactor taxonomy
- High-blast-radius. Always dry-run first: show every file that would move/change and every link that would need rewriting.
- Moving a note: change folder freely (type is in frontmatter). If `type:` changes, re-validate against the new type's required fields.
- Renaming/merging ids: rewrite all inbound wikilinks (the broken-link `rg` recipe must come back clean afterwards). Confirm with the user before applying.

## Output conventions
- Search results: compact list — `title · type/status · id · path`, then the matching snippet.
- Never dump full note bodies unless asked; show the snippet + offer to open.
- For structured rollups, a short table is fine.

## Guardrails
- Decisions (`type: decision`, `status: accepted`) are immutable — supersede with a new ADR linking back, never edit in place.
- Don't create folders beyond the seven top-level folders (five note types plus `sources/` and `inbox/`) without asking.
- Confirm before any operation touching >5 files.
