# Jarvis Index — Storage Model & Metadata Schema

Jarvis is an index of source documents: originals stored verbatim, plus a generated metadata file per source that makes it findable. There is no type/category taxonomy — findability comes from extracted text + SIRA keywords, queried with `rg`.

## Storage layout
```
<kb-root>/
├── sources/<id>/<original-filename>   # verbatim originals, never edited
└── metadata/<id>-<slug>.md            # generated: extracted text + source: + keywords:
```
- `sources/<id>/` keeps the original file with its original filename.
- `metadata/<id>-<slug>.md` is the generated, searchable file — one per source. The same `<id>` pairs a source with its metadata.

## KB-root resolution
1. `$JARVIS_KB` (explicit override).
2. `<git-root>/.jarvis/` when inside a git repo (`git rev-parse --show-toplevel`); ensure `/.jarvis/` is in that repo's `.gitignore`.
3. `~/.jarvis/` (default; create if missing).
4. ask the user.

## Metadata frontmatter
```yaml
---
id: 20260622T150000                 # YYYYMMDDThhmmss; stable, never reused; filename-independent
title: Q3 Financial Report
source:
  path: sources/20260622T150000/q3-report.pdf   # relative to index root
  kind: pdf                         # pdf|docx|xlsx|pptx|csv|html|image|other
  sha256: <hash>                    # integrity + dedup
  ingested: 2026-06-28
  ingester: pdf-text                # extraction path used (see jarvis-index ingestion)
keywords: [k8s, container orchestration]        # SIRA-generated; NOT in the source text
index_generated: 2026-06-28         # when keywords were last generated
---
```

| field | req? | notes |
|-------|------|-------|
| `id` | always | `YYYYMMDDThhmmss`; stable, never reused. |
| `title` | always | human-readable; usually from the source filename/title. |
| `source` | always | block: `path, kind, sha256, ingested, ingester`. |
| `keywords` | recommended | SIRA-generated search terms not present in the source text. |
| `index_generated` | with `keywords` | date keywords were last generated. |

## Body layout
```
<frontmatter>

# <title>

<!-- extracted -->
<verbatim extracted text — headings, tables, page/slide boundaries>
<!-- /extracted -->

<!-- user-notes -->
<!-- /user-notes -->
```
Two fenced regions: extracted text (regenerated on re-index) and user notes (preserved verbatim on re-index). Nothing outside these holds extracted text.

## ID & filename
- `id`: `date +%Y%m%dT%H%M%S` (or a time tool if available).
- filename: `metadata/<id>-<slug>.md`, slug from the title (lowercase, hyphenated, ≤60 chars).
- `sources/<id>/` and `metadata/<id>-slug.md` share the same `id`.

## Re-index / staleness
If the original changed (sha mismatch), re-extract and rewrite **only the `<!-- extracted -->` region**, refresh `source.sha256` + `source.ingested`, regenerate `keywords:` + `index_generated:`, and preserve `id` and the entire `<!-- user-notes -->` region.
