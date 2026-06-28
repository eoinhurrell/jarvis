---
name: jarvis-index
description: Index a source document into the Jarvis source index, or remove one. Use when the user wants to ADD or REMOVE indexed sources — "index this PDF/docx/folder into Jarvis", "add this to the index", "re-index this", "remove/delete that source from Jarvis". Each source is stored verbatim under sources/<id>/ and a generated metadata/<id>-slug.md makes it findable (extracted text + a link back to the source + SIRA keywords). To find sources use jarvis-search; to audit/repair the index use jarvis-doctor.
license: MIT
---

# Jarvis Index — index and remove sources

Add source documents to the Jarvis index, or remove them. This skill **writes** the index; to find sources use `jarvis-search`, to audit/fix use `jarvis-doctor`.

## KB root resolution

Resolve the index root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (explicit override).
2. **Inside a git repo** — `<git-root>/.jarvis/` (`git rev-parse --show-toplevel`); ensure `/.jarvis/` is in that repo's `.gitignore` (append if missing).
3. **`~/.jarvis/`** (default; create if missing).
4. If none is suitable, ask the user.

All paths below are relative to the index root.

## Storage layout

```
<kb-root>/
├── sources/<id>/<original>     # verbatim originals
└── metadata/<id>-<slug>.md     # generated: extracted text + source: + keywords:
```

## Metadata file schema

```yaml
---
id: 20260622T150000
title: Q3 Financial Report
source:
  path: sources/20260622T150000/q3-report.pdf
  kind: pdf            # pdf|docx|xlsx|pptx|csv|html|image|other
  sha256: <hash>
  ingested: 2026-06-28
  ingester: pdf-text
keywords: [k8s, container orchestration]   # SIRA-generated
index_generated: 2026-06-28
---
```
Body: an `<!-- extracted -->` region (verbatim text) and a `<!-- user-notes -->` region (human annotation, preserved on re-index).

## Workflows

### Index a source
1. **Dedup first.** `sha256sum <file>`; if that hash already appears in any metadata file's `source.sha256` (`rg -l '^[[:space:]]*sha256:[[:space:]]*<hash>' -g '*.md' metadata/`), skip and point at the existing source.
2. **Generate the id.** `date +%Y%m%dT%H%M%S` (or a time tool). Copy the original verbatim into `sources/<id>/`, keeping its filename.
3. **Extract faithful text** per `references/ingestion.md` (CLI tools + Python libs; no helper script). Preserve structure so snippets are citeable. Record the ingester.
4. **Write the metadata file** at `metadata/<id>-<slug>.md`: frontmatter above, extracted text in the `<!-- extracted -->` region, an empty `<!-- user-notes -->` region.
5. **Generate SIRA keywords** from the extracted text (`references/sira-index.md`) and fill `keywords:` + `index_generated:`.
6. Never summarize during indexing — keep extracted text verbatim so `rg` hits land.

### Re-index a changed source
If the source changed (sha mismatch), re-extract and rewrite **only the `<!-- extracted -->` region**, refresh `source.sha256`/`source.ingested`, regenerate `keywords:` + `index_generated:`, preserve `id` and the `<!-- user-notes -->` region.

### Remove a source
- Delete `sources/<id>/` and the matching `metadata/<id>*.md`.
- Removal is destructive — confirm before removing >5 sources, and offer to re-index instead where the source still exists elsewhere.

## Where to look
- `references/ingestion.md` — storage model, `source:` contract, per-format extraction recipes.
- `references/sira-index.md` — SIRA keyword generation (index-time prompt, storage, regen rules).
- Find sources → `jarvis-search`; audit/fix → `jarvis-doctor`.
