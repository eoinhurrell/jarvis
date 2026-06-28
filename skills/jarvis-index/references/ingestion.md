# Jarvis Indexing — sources → searchable metadata

Jarvis searches **only the generated markdown** in `metadata/`. Any source file (PDF, Office, HTML, images, code, text, etc.) is *indexed*: the original is stored verbatim under `sources/`, and a metadata markdown file is generated that (a) holds the extracted text so `rg` can find it, and (b) points back to the original so Jarvis can hand the user the real file when that source turns up in a result.

There is **no helper script**. You run extraction directly with CLI tools and Python libraries, then write the metadata file yourself. The recipes below are the per-format steps.

## Storage model

```
<kb-root>/
├── sources/                     # originals, stored verbatim, never edited
│   └── <id>/<original-filename> # e.g. sources/20260622T150000/q3-report.pdf
└── metadata/                    # the generated markdown, one per source
    └── <id>-<slug>.md           # e.g. metadata/20260622T150000-q3-report.md
```

`sources/<id>/` and `metadata/<id>-slug.md` share the same `id`, so a metadata file always pairs with its original.

## Index procedure (run directly — no script)

1. **Dedup first.** Hash the candidate: `sha256sum <file>`. If that hash already appears in any metadata file's `source.sha256` (`rg -l '^[[:space:]]*sha256:[[:space:]]*<hash>' -g '*.md' metadata/`), skip indexing and point at the existing source.
2. **Generate the id.** `date +%Y%m%dT%H%M%S` (or a time tool if available). Create `sources/<id>/` and copy the original in verbatim, keeping its filename.
3. **Extract faithful text** using the per-format recipe below. Preserve structure (headings, tables, page/slide boundaries) so snippets are citeable. Record which ingester path you used.
4. **Write the metadata file** at `metadata/<id>-<slug>.md` with the frontmatter contract below, an `<!-- extracted -->` fenced region holding the verbatim extracted text, and an (initially empty) `<!-- user-notes -->` fenced region for later human annotation. Keeping these regions separate is what makes re-index safe (see "Re-index / staleness").
5. **Generate SIRA keywords** from the extracted text (`references/sira-index.md`) and add `keywords:` + `index_generated:`.
6. **Lint:** eyeball the frontmatter — `id`, `title`, and the `source:` block are all present.

## The source-link contract (how "find the metadata → return the original" works)

Every metadata file carries a `source:` block in frontmatter:

```yaml
---
id: 20260622T150000
title: Q3 Financial Report
source:
  path: sources/20260622T150000/q3-report.pdf   # relative to index root
  kind: pdf            # pdf|docx|xlsx|pptx|csv|html|image|other
  sha256: <hash>       # integrity + dedup
  ingested: 2026-06-28
  pages: 14            # optional, kind-specific metadata
  ingester: pdf-text   # which extraction path was used (see below)
keywords: [k8s, container orchestration]
index_generated: 2026-06-28
---
```

**Resolution rule:** when a metadata file appears in a search result, read its `source.path` (relative to the index root) and present *that* file to the user (via your harness's file-presentation tooling if available), alongside the metadata.

## Body layout

The metadata body has two fenced regions. Extraction text goes in the first; the second is reserved for the user to annotate, and re-index preserves it verbatim. Nothing outside these regions should hold extracted text.

```
<frontmatter>

# <title>

<!-- extracted -->
<verbatim extracted text — headings, tables, page/slide boundaries>
<!-- /extracted -->

<!-- user-notes -->
<!-- /user-notes -->
```

## Per-format extraction

Pick the highest-fidelity path that succeeds; record which one in `source.ingester`. Try text extraction first; fall back to OCR only for pages/sheets/slides that yield little text.

### PDF — `ingester: pdf-text` / `pdf-ocr`
Try text extraction first: `pdftotext -layout <file> -`, or `pdfplumber` in Python. If a page yields little/no text (scanned), rasterize and OCR (`ocrmypdf` or `pytesseract`). Preserve page boundaries as `## Page N` headings so snippets cite a page. Record `pages:`.

### Word — `ingester: docx`
Extract with `pandoc -f docx -t markdown <file>` (preferred) or `python-docx`. Keep paragraphs, headings (as markdown `#`), tables (as markdown tables), and list structure. Drop binary images but note `![image]` placeholders with alt text if present.

### Excel / CSV — `ingester: xlsx` / `csv`
For `.xlsx/.xlsm` use `python` + `openpyxl` (or `ssconvert` / `xlsx2csv`). One `## <SheetName>` section per sheet. Render each sheet as a markdown table **only if small** (≤ ~50 rows × ~15 cols); otherwise emit a header-row + column summary + first/last N rows, and note full data lives in the source. For wide/numeric sheets also emit a plain-text "key: value" flattening of header→cell for the first rows so `rg` can hit specific values. CSV: parse with `csvkit` / Python `csv`, same rules.

### PowerPoint — `ingester: pptx`
Extract with `python-pptx` (or `libreoffice --headless --convert-to` text). One `## Slide N` per slide; slide title as the heading, body text and speaker notes beneath (notes under a `> notes:` blockquote so they're searchable but distinguishable).

### HTML — `ingester: html`
Strip boilerplate and convert to markdown with `pandoc -f html -t markdown` or `html2text`. Preserve headings/links/tables. Keep the source URL in the metadata if remote (and still cache a copy under `sources/`).

### Images — `ingester: image-ocr` / `image-caption`
OCR for text-bearing images (`pytesseract`). For diagrams/photos with little text, write a one-line caption and let SIRA keywords carry findability. The image itself is the resolvable original.

### Code / plain text — `ingester: text`
Copy verbatim into the `<!-- extracted -->` region (optionally fenced in a language code block per file type). Code is already searchable text; SIRA keywords add the vocabulary callers might use ("auth middleware", "rate limiter") that the code doesn't name.

### Anything else — `ingester: other`
Store verbatim under `sources/`, emit a stub metadata file with title, kind, and any filesystem metadata. At minimum the filename and path are searchable.

## Extraction quality markers
- If extraction is partial/low-confidence, add tag-free note in the `<!-- user-notes -->` region and lean on SIRA `keywords:` for findability.
- Always keep the extracted text faithful — don't summarize during indexing (summaries lose `rg` hits).
- Large extractions: keep the full text in the `<!-- extracted -->` region even if long; markdown is cheap and `rg` is fast. Don't truncate searchable content.

## Re-index / staleness
If the original changes (sha mismatch on re-hash), re-extract and rewrite **only the `<!-- extracted -->` region**, refresh `source.sha256` + `source.ingested`, regenerate `keywords:` + `index_generated:`, and preserve `id` and the entire `<!-- user-notes -->` region verbatim. The fenced-region layout is what makes this safe — never re-extract into the user-notes region, and never blow away body content outside the extracted fence.
