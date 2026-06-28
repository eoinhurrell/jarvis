# Jarvis

A collection of installable [AI agent skills](https://agentskills.io/) that **index your source documents and make them findable** — rg-driven, with SIRA keyword retrieval. Packaged in the [vercel-labs/skills](https://github.com/vercel-labs/skills) format, so any of 70+ agent harnesses (Claude Code, Cursor, GitHub Copilot, OpenCode, and many more) can install them with a single command.

> Install from **`eoinhurrell/jarvis`** on GitHub.

## Skills

| Skill | Summary |
|-------|---------|
| [**jarvis**](./skills/jarvis/) | Overview + router. Owns the source-index model (root resolution, the two-folder layout, the metadata schema) and routes to the sub-skills. |
| [**jarvis-search**](./skills/jarvis-search/) | Find indexed sources — ranked full-text + SIRA keyword-field match, default-on query expansion, resolves every hit to its original file. |
| [**jarvis-index**](./skills/jarvis-index/) | Index a source (store the original + generate findable metadata + SIRA keywords), or remove one. |
| [**jarvis-doctor**](./skills/jarvis-doctor/) | Audit and repair index health — missing/stale/un-indexed/duplicate sources, stale keywords — with confirm gates and per-fix verification. |

## Install

Install uses the [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI (`skills`, run via `npx`). It auto-detects the agent harnesses on your machine and installs skills into them (symlink by default, so a single source of truth updates everywhere).

```bash
# Preview the skills this repo provides (no install side effects)
npx skills add eoinhurrell/jarvis --list

# Interactive install — pick skills and target harnesses
npx skills add eoinhurrell/jarvis

# Install just one skill, globally, into Claude Code, non-interactively
npx skills add eoinhurrell/jarvis \
  --skill jarvis-search -a claude-code -g -y

# Install all skills into all detected harnesses
npx skills add eoinhurrell/jarvis --all -y
```

Useful flags:
- `--skill <name>` — install a specific skill (repeatable).
- `-a <agent>` — target a specific harness, e.g. `-a claude-code`, `-a cursor`, `-a opencode` (repeatable). Omit to auto-detect.
- `-g` — install globally (available across all projects) instead of into the current project.
- `-y` — non-interactive; accept defaults (CI-friendly).
- `--all` — install every skill in the repo.

### Install from a local checkout (development)

```bash
git clone https://github.com/eoinhurrell/jarvis.git
cd jarvis
npx skills add .                # installs from the working tree
```

Updating an existing install: `npx skills update [skills…]`.

## The Jarvis skill family

Jarvis is a **local index of source documents**. You point it at files (PDFs, Office docs, code, text); it stores each original verbatim and generates a searchable metadata file so you can find the source again. No database, no daemon: every query walks the tree live with `ripgrep`, and findability is a **derived view** computed on demand from the generated metadata and [SIRA](https://arxiv.org/pdf/2605.06647) keywords. This keeps the index portable, harness-agnostic, and impossible to corrupt.

The index is **two folders**:
- `sources/<id>/<original>` — the files you asked to index, verbatim.
- `metadata/<id>-<slug>.md` — the generated, searchable file: extracted text (so `rg` hits), a `source:` link back to the original, and SIRA `keywords:` (vocabulary a user might search with that *isn't* in the source).

It's managed by four focused, **independently-installable** skills: [`jarvis`](./skills/jarvis/) (overview/router), [`jarvis-search`](./skills/jarvis-search/) (find), [`jarvis-index`](./skills/jarvis-index/) (index/remove), [`jarvis-doctor`](./skills/jarvis-doctor/) (audit/repair).

**Where the index lives** (resolved in this order):
1. `$JARVIS_KB` env var (override)
2. `<git-root>/.jarvis/` when invoked inside a git repo (auto-added to the repo's `.gitignore`)
3. `~/.jarvis/` (the default; created on first use)

### Runtime prerequisites

- **Required:** [`ripgrep`](https://github.com/BurntSushi/ripgrep) (`rg`) — every query depends on it.
- **Optional** (only for indexing non-text sources, via `jarvis-index`): `pdftotext`/`ocrmypdf`/`pytesseract` (PDF/OCR), `pandoc` (Word/HTML), `python-docx` / `openpyxl` / `python-pptx` (Office), `csvkit` (CSV).

## Repository layout

```
jarvis/
├── README.md
├── CLAUDE.md
├── LICENSE
└── skills/
    ├── jarvis/                  # overview + router (canonical model)
    │   ├── SKILL.md
    │   └── references/schema.md
    ├── jarvis-search/           # find sources
    │   ├── SKILL.md
    │   └── references/{search-recipes.md, query-expansion.md}
    ├── jarvis-index/            # index / remove sources
    │   ├── SKILL.md
    │   └── references/{ingestion.md, sira-index.md}
    └── jarvis-doctor/           # audit + repair
        ├── SKILL.md
        └── references/diagnostics.md
```

Each skill is a directory under `skills/` containing a `SKILL.md` (YAML frontmatter + instructions) plus optional `references/`. See [`CLAUDE.md`](./CLAUDE.md) for how to add and validate a skill.

## Adding a skill

See [`CLAUDE.md`](./CLAUDE.md) for the `SKILL.md` spec, layout conventions, and local validation steps.

## License

[MIT](./LICENSE).
