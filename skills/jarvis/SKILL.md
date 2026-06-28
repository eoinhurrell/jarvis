---
name: jarvis
description: Overview and router for the Jarvis source index — a local, rg-driven index of your source documents (no database, no daemon; findability is derived on demand from generated metadata + SIRA keywords). Use this skill when the user refers to Jarvis or the index by name and the request spans the whole index or you're unsure which sub-skill fits — "what's in the Jarvis index", "set up Jarvis", "where does Jarvis store things". It owns the index model (root resolution, the two-folder layout, the metadata schema) and routes focused work to jarvis-search (find sources), jarvis-index (index/remove sources), and jarvis-doctor (audit/repair). Do NOT trigger on generic note-taking ("add a note", "my notes") — those are better served by a general note-taking/Zettelkasten tool.
license: MIT
---

# Jarvis — Local Source Index

Jarvis is a local index of your **source documents**. You point it at files (PDFs, Office docs, code, text); it stores each original verbatim and generates a searchable metadata file so you can find the source again. No database, no daemon: every query walks the tree live with `rg`, and findability is a **derived view** computed on demand from the generated metadata and SIRA keywords. This keeps the index portable, harness-agnostic, and impossible to corrupt.

## The Jarvis skill family

Jarvis is split into focused, **independently-installable** skills. This `jarvis` skill is the overview and router — it owns the model below. Install the ones you need:

| Skill | Does | Triggers |
|-------|------|----------|
| **jarvis-search** | Find sources (full-text + keyword match, with SIRA query expansion) | "jarvis, find…", "search the index for…" |
| **jarvis-index** | Index a source, or remove one | "index this PDF", "add this to Jarvis", "remove that source" |
| **jarvis-doctor** | Audit and repair index health | "check the Jarvis index", "find stale sources", "dedupe" |

Because each skill installs on its own, every skill carries its own copy of the index contract. **This skill is the canonical source** — keep the others in sync with it.

## KB root resolution

Resolve the index root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (explicit override).
2. **Inside a git repo** — `<git-root>/.jarvis/`, where `<git-root>` is `git rev-parse --show-toplevel`. Ensure `/.jarvis/` is in that repo's `.gitignore` (append it if missing) so the index isn't committed.
3. **`~/.jarvis/`** (default; create if missing).
4. If none is suitable, ask the user and suggest exporting `$JARVIS_KB`.

All paths below are relative to the index root.

## Storage layout

The index is just two folders:
- **`sources/<id>/<original-filename>`** — the files you asked to index, stored verbatim.
- **`metadata/<id>-<slug>.md`** — the generated, searchable file that finds its source: extracted text (so `rg` hits), a `source:` block pointing back to the original, and SIRA `keywords:`.

```
<kb-root>/
├── sources/<id>/<original>     # verbatim originals
└── metadata/<id>-<slug>.md     # generated: extracted text + source: + keywords:
```

## Metadata file schema

```yaml
---
id: 20260622T150000                 # YYYYMMDDThhmmss; stable, never reused
title: Q3 Financial Report
source:
  path: sources/20260622T150000/q3-report.pdf   # relative to index root
  kind: pdf                         # pdf|docx|xlsx|pptx|csv|html|image|other
  sha256: <hash>                    # integrity + dedup
  ingested: 2026-06-28
  ingester: pdf-text                # extraction path used
keywords: [k8s, container orchestration]        # SIRA-generated; NOT in the source text
index_generated: 2026-06-28         # when keywords were last generated
---
```
The body holds an `<!-- extracted -->` region (verbatim extracted text) and a `<!-- user-notes -->` region (human annotation, preserved on re-index).

Full storage model and metadata contract: `references/schema.md`.

## Routing

- Want to **find** a source? → `jarvis-search`
- **Index** a source, or **remove** one? → `jarvis-index`
- **Audit / repair** the index? → `jarvis-doctor`

If only this `jarvis` skill is installed, you can still answer using the model above and plain `rg`, but the focused skill gives the tuned workflow.

## Do NOT use for

- Generic note-taking — "add a note", "my notes", "look this up in my notes", "is there anything on…" → use a general note-taking tool.
- Atomic / permanent / literature notes, or building a knowledge graph → use a dedicated Zettelkasten tool.

## Guardrails

- Confirm before any operation touching >5 files.
- Re-indexing rewrites only the `<!-- extracted -->` region; never blow away the `<!-- user-notes -->` region.
