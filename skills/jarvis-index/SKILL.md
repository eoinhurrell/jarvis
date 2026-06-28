---
name: jarvis-index
description: Add, update, ingest, or remove notes in the Jarvis local knowledge base of markdown notes. Use when the user wants to CAPTURE or CHANGE KB content — "add this to the KB", "create a note for…", "ingest this PDF/docx into Jarvis", "update that note", "archive/remove/delete this note". Notes use timestamp ids and YAML frontmatter by type (project/org/team/reference/decision); non-markdown files are ingested into searchable markdown linking back to the stored original. Removal defaults to archiving; hard-delete requires confirmation and checks inbound links. To search use jarvis-search; to audit/fix use jarvis-doctor.
license: MIT
---

# Jarvis Index — add, update, ingest, and remove notes

Capture and change content in the Jarvis knowledge base. This skill **writes**; to find notes use `jarvis-search`, to audit/fix use `jarvis-doctor`.

## KB contract (compact)

**Root** — `$JARVIS_KB` → a `.jarvis` marker (walk up from CWD) → `~/.jarvis` (default; create if missing) → ask. Paths below are relative to the KB root.

**rg-first, no index.** Type lives in frontmatter (`type:`); folders are shallow. Cross-cutting via `tags:` + `[[wikilinks]]`.

**Types & folders:** `projects/ project · org/ org · teams/ team · reference/ reference · decisions/ decision · sources/ (ingested originals) · inbox/ (capture)`.

**Schema (common fields):**
```yaml
---
id: 20260622T143000     # YYYYMMDDThhmmss; stable, never reused
title: Human Readable Title
type: project|org|team|reference|decision
status: draft|active|blocked|done|accepted|superseded|archived
project: project-slug
team: team-slug
owner: name-or-handle
created: 2026-06-22
updated: 2026-06-22
tags: [retrieval, mlops]
---
```
Outbound links are inline `[[id-or-slug]]` in the body — no `links:` array. Authoritative per-type required fields and the ADR lifecycle: `references/taxonomy.md`.

## Workflows

### Create a note
1. Read `references/taxonomy.md` for the type's required fields.
2. Generate `id` from the current timestamp (use the `date` command, or a time MCP tool if available); filename `<id>-<slug>.md`.
3. Place in the type's folder (or `inbox/` to triage later).
4. Fill frontmatter per schema; set `created:` and `updated:` to today.
5. Lint: eyeball required fields present, valid `type`/`status`.

### Update a note
- Bump `updated:`; never rewrite `id` or `created`.
- Re-validate against the type's required fields if `type:` changed.

### Ingest a non-markdown document (PDF, Word, Excel, PPT, CSV, HTML, images)
Jarvis searches markdown only, so other formats become a derived markdown note that links back to the stored original. **Read `references/ingestion.md` first** — storage model, the `source:` frontmatter contract, and per-format extraction steps (CLI tools + Python libs; no helper script).
- Hash-and-dedup → generate `id` → copy original verbatim into `sources/<id>/` → extract faithful text → write a derived note with a `source:` block (path + sha256 + kind + ingester).
- Never summarize during ingestion — keep extracted text verbatim so `rg` hits land.

### Remove a note
Jarvis prefers **archiving** over destructive deletes.
- **Archive (default):** set `status: archived` and keep the file. Use this unless the user clearly wants it gone. Reversible and link-safe.
- **Hard-delete (explicit confirm only):**
  1. Before deleting, find **inbound links** — notes whose body references this note's `id` as `[[id]]` — and warn the user; offer to rewrite or remove those links (otherwise they break).
  2. If the note has a `source:` block, also remove its `sources/<id>/` original directory.
  3. Delete the `.md` file.
  4. **Guardrail:** never hard-delete an accepted decision (`type: decision`, `status: accepted`) — supersede it with a new ADR instead.
- Confirm before removing >5 files.

## Where to look
- `references/taxonomy.md` — full schema, per-type required fields, status lifecycles.
- `references/ingestion.md` — ingestion contract + per-format recipes.
- Search → `jarvis-search`; audit/fix → `jarvis-doctor`.
