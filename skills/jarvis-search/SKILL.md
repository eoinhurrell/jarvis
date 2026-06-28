---
name: jarvis-search
description: Find indexed sources in the Jarvis source index. Use when the user wants to FIND a source document — "jarvis, find…", "search the index for…", "where's the doc about X", "what did we index that mentions…". Searches the generated metadata/ with ripgrep (ranked full-text + SIRA keyword-field matching), expands the query with SIRA vocabulary by default, and resolves every hit back to its original source file. To index or remove sources use jarvis-index; to audit/repair the index use jarvis-doctor.
license: MIT
---

# Jarvis Search — find indexed sources

Find source documents in the Jarvis index. No database, no daemon: every query walks `metadata/` live with `rg`. This skill only **reads** — to index/remove sources use `jarvis-index`, to audit/fix use `jarvis-doctor`.

## KB root resolution

Resolve the index root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (explicit override).
2. **Inside a git repo** — `<git-root>/.jarvis/` (`git rev-parse --show-toplevel`); ensure `/.jarvis/` is in that repo's `.gitignore`.
3. **`~/.jarvis/`** (default; create if missing).
4. If none is suitable, ask the user.

All paths below are relative to the index root.

## Storage layout (compact)

- `metadata/<id>-<slug>.md` — generated, searchable: extracted text + a `source:` link + `keywords:` (SIRA).
- `sources/<id>/<original>` — the verbatim original a metadata file resolves back to.

Every metadata file has a `source:` block, so every hit resolves to a real source.

## Search workflow

1. **Expand the query (default-on).** For a natural-language query, generate extra search terms via the SIRA query-expansion prompt (`references/query-expansion.md`), drop any term hitting zero metadata files (DF>0 filter), and rank by weighted match density so original terms outrank expansion-only hits. Skip expansion for precise lookups (a source `id`, an exact quoted phrase, a filename) or when the user opts out.
2. **Compose the query.** For anything beyond a plain keyword, read `references/search-recipes.md` first — ranked full-text, `^keywords:` field matching, and weighted-density ranking. (Expanded terms join these patterns; sources carrying matching `keywords:` surface here.)
3. **Full-text** — `rg -i` over `metadata/`; rank by match density.
4. **Always surface `id` and path** so the result can be opened.
5. **Resolve the source.** Every hit has a `source:` block — read `source.path` (relative to the index root) and present that original file to the user (via your harness's file-presentation tooling if available), not just the metadata.

## Output conventions
- Compact list — `title · id · path`, then the matching snippet (`rg -C1`).
- Never dump full metadata bodies unless asked; show the snippet + offer to open the source.
- Note briefly whether each top hit matched on original text or an expanded/SIRA keyword.

## Where to look
- `references/search-recipes.md` — `rg` patterns: ranked full-text, `keywords:` matching, weighted ranking.
- `references/query-expansion.md` — SIRA query expansion (query-time prompt, DF>0 filter, weighted ranking).
- Index/remove sources → `jarvis-index`; audit/fix → `jarvis-doctor`.
