# Jarvis

A collection of installable [AI agent skills](https://agentskills.io/) for managing a local knowledge base and related workflows. Packaged in the [vercel-labs/skills](https://github.com/vercel-labs/skills) format, so any of 70+ agent harnesses (Claude Code, Cursor, GitHub Copilot, OpenCode, and many more) can install them with a single command.

> Install from **`eoinhurrell/jarvis`** on GitHub.

## Skills

| Skill | Summary |
|-------|---------|
| [**jarvis-knowledgebase**](./skills/jarvis-knowledgebase/) | Manage a local corporate knowledge base / second brain of markdown notes — capture, search, organize, ingest non-markdown documents. rg-first, no database, harness-agnostic. |

## Install

Install uses the [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI (`skills`, run via `npx`). It auto-detects the agent harnesses on your machine and installs skills into them (symlink by default, so a single source of truth updates everywhere).

```bash
# Preview the skills this repo provides (no install side effects)
npx skills add eoinhurrell/jarvis --list

# Interactive install — pick skills and target harnesses
npx skills add eoinhurrell/jarvis

# Install just this skill, globally, into Claude Code, non-interactively
npx skills add eoinhurrell/jarvis \
  --skill jarvis-knowledgebase -a claude-code -g -y

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

## jarvis-knowledgebase

Jarvis is a **local corporate second-brain** of plain markdown notes. No database, no daemon: every query walks the tree live with `ripgrep`, and the structured index (by type/project/team/status/tag/link) is a **derived view computed on demand** from YAML frontmatter. That keeps the KB portable, harness-agnostic, and impossible to corrupt.

- **Five note types** — `project`, `org`, `team`, `reference`, `decision` — plus `sources/` (verbatim originals of ingested files) and `inbox/` (capture to triage).
- **Cross-cutting structure** via `tags:` and `[[wikilinks]]`, not deep folders.
- **Non-markdown ingestion** — PDF/Word/Excel/PPT/CSV/HTML/images are converted to searchable markdown that links back to the stored original.
- **Maintenance** — orphan detection, broken-link audits, and frontmatter linting, all from `rg` recipes.

**Where the KB lives** (resolved in this order):
1. `$JARVIS_KB` env var (override)
2. a `.jarvis` marker file (walk up from the current directory — for a project-local KB)
3. `~/.jarvis` (the default; created on first use)

See the skill's own [`SKILL.md`](./skills/jarvis-knowledgebase/SKILL.md) and its [`references/`](./skills/jarvis-knowledgebase/references/) for the full taxonomy, search recipes, and ingestion contract.

### Runtime prerequisites

- **Required:** [`ripgrep`](https://github.com/BurntSushi/ripgrep) (`rg`) — every query depends on it.
- **Optional** (only for ingesting non-markdown files): `pdftotext`/`ocrmypdf`/`pytesseract` (PDF/OCR), `pandoc` (Word/HTML), `python-docx` / `openpyxl` / `python-pptx` (Office), `csvkit` (CSV).

## Repository layout

```
jarvis/
├── README.md
├── CLAUDE.md
├── LICENSE
└── skills/
    └── jarvis-knowledgebase/
        ├── SKILL.md
        └── references/
            ├── taxonomy.md
            ├── search-recipes.md
            └── ingestion.md
```

Each skill is a directory under `skills/` containing a `SKILL.md` (YAML frontmatter + instructions) plus optional `references/`, `scripts/`, or `assets/`. See [`CLAUDE.md`](./CLAUDE.md) for how to add and validate a skill.

## Adding a skill

See [`CLAUDE.md`](./CLAUDE.md) for the `SKILL.md` spec, layout conventions, and local validation steps.

## License

[MIT](./LICENSE).
