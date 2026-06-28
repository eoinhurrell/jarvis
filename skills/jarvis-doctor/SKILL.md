---
name: jarvis-doctor
description: Audit and repair the Jarvis local knowledge base of markdown notes. Use when the user wants to CHECK or FIX KB health — "jarvis, fix the KB", "find orphan notes", "check for broken links", "lint the frontmatter", "clean up the KB", "refactor the taxonomy", "are there stale notes". Diagnoses orphans, broken [[wikilinks]], and frontmatter issues with ripgrep, then proposes and applies fixes behind explicit confirm gates (each fix is verified by re-running the diagnostic until clean). To search use jarvis-search; to add or remove notes use jarvis-index.
license: MIT
---

# Jarvis Doctor — audit and repair the knowledge base

Diagnose and fix common KB issues, then verify each fix. All checks are `rg` over the live tree (no index). This skill **mutates** the KB under confirm gates; to just search use `jarvis-search`, to add/remove notes use `jarvis-index`.

## KB contract (compact)

**Root** — `$JARVIS_KB` → a `.jarvis` marker (walk up from CWD) → `~/.jarvis` (default; create if missing) → ask. Paths relative to KB root.

**rg-first, no index.** Five types in frontmatter (`project|org|team|reference|decision`); cross-cutting via `tags:` + inline `[[wikilinks]]` (no `links:` array). Notes open with YAML (`id, title, type, status, created, updated, …`).

## Diagnostic → fix loop

1. **Run diagnostics** — orphans, broken `[[wikilinks]]`, frontmatter lint. Exact `rg` pipelines and the decision flow (Mermaid) are in `references/diagnostics.md` — read both before acting.
2. **Classify** each finding and **propose** a fix (fix the id / create a stub / remove the link / link the orphan / archive / fill the missing field / correct an invalid type or status).
3. **Confirm gate** — any fix touching >5 files, or any accepted decision, requires showing the full plan and getting explicit confirmation.
4. **Apply** the fix, then **re-run the same diagnostic** — it must come back clean (verification). If not, re-classify and repeat.
5. **Report** what was found, what was fixed, and what was left flagged for the user.

## What it checks
- **Orphans** — notes whose `id` is linked nowhere and that declare no outbound links.
- **Broken links** — `[[id]]` references whose target `id` doesn't exist.
- **Frontmatter lint** — notes missing required fields (`id, title, type, status, created`) or carrying an invalid `type`/`status`; per-type extras from the table in `references/diagnostics.md`.
- **Refactors** — taxonomy moves / id merges (high blast radius): always dry-run first (list every file that moves and every link that needs rewriting), confirm, then apply and verify links are clean.

## Guardrails
- Decisions (`type: decision`, `status: accepted`) are immutable — never edit in place; propose supersession instead.
- Never bulk-rewrite without confirming. Confirm before any operation touching >5 files.
- Don't create folders beyond the seven top-level without asking.

## Where to look
- `references/diagnostics.md` — `rg` pipelines + per-type required-fields table + refactor dry-run + the Mermaid decision diagram.
- Search → `jarvis-search`; add/remove → `jarvis-index`.
