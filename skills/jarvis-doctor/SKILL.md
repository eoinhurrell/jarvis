---
name: jarvis-doctor
description: Audit and repair the Jarvis source index. Use when the user wants to CHECK or FIX index health — "check the Jarvis index", "find stale sources", "find missing sources", "dedupe the index", "re-index what changed", "are there orphaned files". Diagnoses sources whose original is missing or changed, un-indexed sources, stale SIRA keywords, and duplicate sources with ripgrep, then proposes and applies fixes behind explicit confirm gates (each fix is verified by re-running the check until clean). To find sources use jarvis-search; to index or remove sources use jarvis-index.
license: MIT
---

# Jarvis Doctor — audit and repair the source index

Diagnose and fix index-health issues, then verify each fix. All checks are `rg`/shell over `metadata/` and `sources/`. This skill **mutates** the index under confirm gates; to search use `jarvis-search`, to index/remove sources use `jarvis-index`.

## KB root resolution

Resolve the index root in this order, stopping at the first hit:
1. `$JARVIS_KB` env var (explicit override).
2. **Inside a git repo** — `<git-root>/.jarvis/` (`git rev-parse --show-toplevel`); ensure `/.jarvis/` is in that repo's `.gitignore`.
3. **`~/.jarvis/`** (default; create if missing).
4. If none is suitable, ask the user.

All paths below are relative to the index root.

## Storage layout (compact)

- `metadata/<id>-<slug>.md` — generated, with a `source:` block (`path`, `sha256`, …) + `keywords:`.
- `sources/<id>/<original>` — the verbatim originals.

The `<id>` pairs a metadata file with its source.

## Diagnostic → fix loop

1. **Run diagnostics** — missing source, un-indexed source, stale source (sha mismatch), stale keywords, duplicate source. Exact pipelines and the decision flow (Mermaid) are in `references/diagnostics.md` — read both before acting.
2. **Classify** each finding and **propose** a fix (remove orphaned metadata / index the source / re-index the changed source / regenerate keywords / dedupe).
3. **Confirm gate** — any fix touching >5 files requires showing the full plan and getting explicit confirmation.
4. **Apply** the fix, then **re-run the same check** — it must come back clean (verification). If not, re-classify and repeat.
5. **Report** what was found, fixed, and left flagged.

## What it checks
- **Missing source** — a metadata file whose `source.path` no longer exists (original deleted). Offer to remove the orphaned metadata, or re-acquire.
- **Un-indexed source** — a `sources/<id>/` entry with no matching `metadata/<id>*.md`. Offer to index it.
- **Stale source** — `source.sha256` ≠ current hash of the original (source changed). Offer to re-index.
- **Stale keywords** — source changed but `keywords:` not regenerated, or `index_generated` missing. Offer to regenerate via `jarvis-index`'s SIRA step.
- **Duplicate source** — same `sha256` across two metadata files. Offer to dedupe (keep one).

## Guardrails
- Never bulk-rewrite without confirming. Confirm before any operation touching >5 files.
- Re-indexing rewrites only the `<!-- extracted -->` region; preserve `id` and the `<!-- user-notes -->` region.

## Where to look
- `references/diagnostics.md` — `rg`/shell pipelines for each check + the Mermaid decision diagram.
- Search → `jarvis-search`; index/remove → `jarvis-index`.
