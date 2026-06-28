---
name: jarvis-search
description: Find notes in the Jarvis local knowledge base of markdown notes. Use when the user wants to FIND information in the KB — "jarvis, find…", "search the KB for…", "where in the KB did we decide X", "what notes mention…", "list active projects owned by team Y". Returns ranked full-text hits and structured queries (by type/status/team/tag/owner) over YAML frontmatter, driven by ripgrep over the live tree with no persisted index. To add, update, ingest, or remove notes use jarvis-index; to audit or fix the KB use jarvis-doctor.
license: MIT
---

# Jarvis Search — find notes in the knowledge base

Find notes in the Jarvis knowledge base. No database, no daemon: every query walks the tree live with `rg`. This skill only **reads** — to add/remove notes use `jarvis-index`, to audit/fix use `jarvis-doctor`.

## KB contract (compact)

**Root** — resolve in order: `$JARVIS_KB` → a `.jarvis` marker (walk up from CWD) → `~/.jarvis` (default; create if missing) → ask. Paths below are relative to the KB root.

**rg-first, no index** — structured queries are frontmatter `rg` patterns; full-text is body `rg`.

**Types** (`type:` field; cross-cutting via `tags:` + `[[wikilinks]]`):
```
projects/ project · org/ org · teams/ team · reference/ reference · decisions/ decision · sources/ (ingested originals) · inbox/ (capture)
```

**Frontmatter fields:** `id, title, type, status, project, team, owner, created, updated, tags, source`. Outbound links are inline `[[id]]` in the body (no `links:` array). A note with a `source:` block was ingested from a non-markdown original.

## Search workflow

1. **Compose the query.** For anything beyond a plain keyword, read `references/search-recipes.md` first — tuned `rg` patterns for every field, tag membership, ranked full-text, and the multi-field (AND) intersect.
2. **Full-text** — `rg -i` over bodies; rank by match density.
3. **Structured** ("active projects owned by team X", "accepted decisions tagged retrieval") — chain frontmatter `rg` with `-l` + a second pass (recipes in the reference).
4. **Always surface `id` and path** so the result can be opened/linked.
5. **Ingested-source resolution** — if a hit has a `source:` block, read its `source.path` and present that original file to the user (via your harness's file-presentation tooling if available), not just the derived markdown.

## Output conventions
- Compact list — `title · type/status · id · path`, then the matching snippet (`rg -C1`).
- Never dump full note bodies unless asked; show the snippet + offer to open.
- For structured rollups (counts by type/status), a short table is fine.

## Where to look
- `references/search-recipes.md` — every `rg` pattern you need (read before non-trivial queries).
- Add / update / ingest / remove a note → `jarvis-index`.
- Audit / fix / refactor → `jarvis-doctor`.
