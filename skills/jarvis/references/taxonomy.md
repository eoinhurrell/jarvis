# Jarvis Taxonomy & Frontmatter Schema

A shallow tree of five note **types** plus capture/source folders. Type lives in frontmatter (`type:`), so a note can move folders without breaking any query. Cross-cutting structure is carried by `tags:` and wikilinks, never by deeper folders.

```
<kb-root>/
├── projects/    type: project    active, time-bound deliverables
├── org/         type: org        company-wide: policy, structure, OKRs
├── teams/       type: team       pillar/team charters, ownership, runbooks
├── reference/   type: reference  durable evergreen knowledge, how-tos
├── decisions/   type: decision   ADR-style records; immutable once accepted
├── sources/     —                verbatim originals of ingested files
└── inbox/       —                untyped capture, triage into a type later
```

## Frontmatter fields

| field    | req?            | notes |
|----------|-----------------|-------|
| `id`     | always          | stable, never reused; `YYYYMMDDThhmmss`. Filename-independent. |
| `title`  | always          | human-readable. |
| `type`   | always          | one of the five types. |
| `status` | always          | see lifecycle per type below. |
| `created`| always          | ISO date, set once. |
| `updated`| always          | ISO date, bump on every edit. |
| `tags`   | recommended     | `[a, b]`; cross-cutting topics. |
| `project`| if relevant     | project slug this note belongs to. |
| `team`   | project/team/org| owning pillar/team slug. |
| `owner`  | project/team    | person/handle accountable. |
| `source` | ingested only   | block: `path, kind, sha256, ingested, ingester` (+ type-specific meta). |
| `keywords` | optional (SIRA) | `[a, b, c]` — LLM-generated search terms NOT present in the note text; bridge vocabulary mismatch. Hit by both full-text `rg` and `^keywords:` queries. |
| `index_generated` | with `keywords` | ISO date the keywords were last generated; regenerate when the body changes. |

## Status lifecycle by type

- **project**: `draft → active → blocked → done → archived`
- **org**: `draft → active → archived`
- **team**: `draft → active → archived`
- **reference**: `draft → active → archived` (use `archived` when superseded by newer reference)
- **decision (ADR)**: `proposed → accepted → superseded`. **Accepted decisions are immutable.** To change one, write a new decision with `status: accepted` that links back with an inline `[[old-id]]` wikilink, and set the old note to `status: superseded` with a forward `[[new-id]]` link in its body. Never edit an accepted decision's substance in place. (Outbound links are always inline `[[id]]` in the body — there is no `links:` array.)

## Required fields per type (beyond the always-required id/title/type/status/created/updated)

- **project**: `team`, `owner`; `project` slug should match the folder/grouping.
- **team**: `owner` (lead).
- **org**: none extra.
- **reference**: none extra.
- **decision**: `owner` (decider) recommended; body should carry Context / Decision / Consequences sections.

## ID & filename conventions
- `id`: `date +%Y%m%dT%H%M%S` at creation (or a time tool if available).
- filename: `<id>-<slug>.md`, slug from the title (lowercase, hyphenated, ≤60 chars).
- Renaming the file is safe; renaming the `id` requires rewriting inbound wikilinks (see search-recipes maintenance).

## Tag conventions
- lowercase, hyphenated, singular where natural (`retrieval`, `mlops`, `needs-review`).
- `needs-review` / `needs-description` are reserved hygiene tags set by ingestion when extraction confidence is low.

## Refactor rules
- Moving a note between folders: free (type is in frontmatter). If `type:` itself changes, re-validate required fields for the new type.
- Always dry-run a multi-file refactor: list every file that moves and every link that needs rewriting, confirm, then apply.
- Never create folders beyond the seven above without asking the user.
