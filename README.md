# Jarvis

A collection of installable [AI agent skills](https://agentskills.io/) for managing a local knowledge base and related workflows. Packaged in the [vercel-labs/skills](https://github.com/vercel-labs/skills) format, so any of 70+ agent harnesses (Claude Code, Cursor, GitHub Copilot, OpenCode, and many more) can install them with a single command.

> Install from **`eoinhurrell/jarvis`** on GitHub.

## Skills

| Skill | Summary |
|-------|---------|
| [**jarvis**](./skills/jarvis/) | Overview + router. Owns the KB model (root, taxonomy, schema) and routes to the focused skills below. |
| [**jarvis-search**](./skills/jarvis-search/) | Find notes — ranked full-text + structured (by type/status/team/tag) queries over YAML frontmatter. rg-driven, read-only. |
| [**jarvis-index**](./skills/jarvis-index/) | Add, update, ingest, or remove notes. Timestamp ids, frontmatter by type; non-markdown ingestion; archive-by-default removal. |
| [**jarvis-doctor**](./skills/jarvis-doctor/) | Audit and repair the KB — orphans, broken `[[wikilinks]]`, frontmatter lint, refactors — with confirm gates and per-fix verification. |

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

Jarvis is a **local corporate second-brain** of plain markdown notes. No database, no daemon: every query walks the tree live with `ripgrep`, and the structured index (by type/project/team/status/tag/link) is a **derived view computed on demand** from YAML frontmatter. That keeps the KB portable, harness-agnostic, and impossible to corrupt.

The KB is managed by four focused, **independently-installable** skills (install only what you need):

- **[`jarvis`](./skills/jarvis/)** — overview + router. Owns the KB model: root resolution, the five note types (`project`, `org`, `team`, `reference`, `decision`) plus `sources/` (verbatim originals) and `inbox/` (capture), the frontmatter schema, and the `tags:` + `[[wikilinks]]` cross-cutting model.
- **[`jarvis-search`](./skills/jarvis-search/)** — find notes (read-only): ranked full-text + structured queries.
- **[`jarvis-index`](./skills/jarvis-index/)** — add / update / ingest / remove notes; non-markdown files become searchable markdown linking back to the original.
- **[`jarvis-doctor`](./skills/jarvis-doctor/)** — audit and repair: orphans, broken links, frontmatter lint, refactors.

**Where the KB lives** (resolved in this order):
1. `$JARVIS_KB` env var (override)
2. a `.jarvis` marker file (walk up from the current directory — for a project-local KB)
3. `~/.jarvis` (the default; created on first use)

Each skill's `SKILL.md` and `references/` carry the full detail; the `jarvis` umbrella is the canonical KB model.

### Runtime prerequisites

- **Required:** [`ripgrep`](https://github.com/BurntSushi/ripgrep) (`rg`) — every query depends on it.
- **Optional** (only for ingesting non-markdown files, via `jarvis-index`): `pdftotext`/`ocrmypdf`/`pytesseract` (PDF/OCR), `pandoc` (Word/HTML), `python-docx` / `openpyxl` / `python-pptx` (Office), `csvkit` (CSV).

## Repository layout

```
jarvis/
├── README.md
├── CLAUDE.md
├── LICENSE
└── skills/
    ├── jarvis/                  # overview + router (canonical KB model)
    │   ├── SKILL.md
    │   └── references/taxonomy.md
    ├── jarvis-search/           # find notes (read-only)
    │   ├── SKILL.md
    │   └── references/search-recipes.md
    ├── jarvis-index/            # add / update / ingest / remove
    │   ├── SKILL.md
    │   └── references/{taxonomy.md, ingestion.md}
    └── jarvis-doctor/           # audit + repair
        ├── SKILL.md
        └── references/diagnostics.md
```

Each skill is a directory under `skills/` containing a `SKILL.md` (YAML frontmatter + instructions) plus optional `references/`, `scripts/`, or `assets/`. See [`CLAUDE.md`](./CLAUDE.md) for how to add and validate a skill.

## Adding a skill

See [`CLAUDE.md`](./CLAUDE.md) for the `SKILL.md` spec, layout conventions, and local validation steps.

## License

[MIT](./LICENSE).
