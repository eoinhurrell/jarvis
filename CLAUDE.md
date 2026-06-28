# CLAUDE.md

Guidance for AI agents (and humans) working **in this repository**.

## What this repo is

A collection of [AI agent skills](https://agentskills.io/) in the [vercel-labs/skills](https://github.com/vercel-labs/skills) format. Consumers install them with the `skills` CLI:

```bash
npx skills add eoinhurrell/jarvis          # interactive
npx skills add eoinhurrell/jarvis --list   # preview, no install
npx skills add .                               # from a local checkout
```

The repo itself contains **no application code** — only skill definitions. Don't add a build step, runtime, or package.json unless a skill genuinely needs bundled scripts.

## Layout convention

```
skills/
└── <skill-name>/
    ├── SKILL.md            # required: frontmatter + instructions
    ├── references/         # optional: long-form detail loaded on demand
    ├── scripts/            # optional: executable helpers
    └── assets/             # optional: templates, data
```

The `skills` CLI discovers skills 1–2 levels deep under `skills/`. One directory per skill.

## SKILL.md spec (the hard rules)

Frontmatter is **strictly validated**. Get these right or the skill won't install/discover:

- **`name`** (required) — must **equal the directory name**. Lowercase letters, digits, and hyphens only; 1–64 chars; no leading/trailing hyphen; no consecutive hyphens (`--`).
- **`description`** (required) — 1–1024 chars. Say **what** the skill does **and when to use it**, including trigger phrases an agent can match on.
- Optional: `license`, `compatibility` (≤500 chars), `metadata` (key/value), `allowed-tools`.

Body conventions (progressive disclosure):
- Keep `SKILL.md` under ~500 lines. Put long detail in `references/` and reference it by relative path — those files load only when the skill needs them.
- Reference files with paths relative to the skill root, e.g. `references/schema.md`.
- Be harness-agnostic. Don't hard-depend on a specific sibling skill or a named MCP tool; phrase such things as "if available" / "use a dedicated X tool".

## Adding a skill

1. `mkdir skills/<kebab-case-name>` (must match the `name` field).
2. Write `SKILL.md` with valid frontmatter (`name`, `description`, optional `license: MIT`).
3. Move any long content into `references/`.
4. Add a row to the skills table in `README.md`.
5. Validate (below).

## Validate / test locally

```bash
# Confirm the CLI discovers every skill under skills/ (no install side effects)
npx skills add . --list

# If the skills-ref tool is installed, validate frontmatter
skills-ref validate skills/<name>
```

Manual checks before committing:
- The `name` field equals the directory name; no `--`, ≤64 chars.
- `description` ≤ 1024 chars.
- Every `references/...` path mentioned in `SKILL.md` resolves.
- For any `jarvis-*` skill: no leftover references to plugin-specific tools/skills — `rg -n "user_time_v0|present_files" skills/` should return nothing.

A full test install (writes into the target harness): `npx skills add . --skill <name> -y`.

## The Jarvis skill family

Jarvis is a **source index** (not a note taxonomy): originals stored verbatim under `sources/<id>/`, with a generated `metadata/<id>-slug.md` per source (extracted text + a `source:` link + SIRA `keywords:`) that makes it findable. Four independently-installable skills:

- **`jarvis`** — overview + router; owns the **canonical model** (`skills/jarvis/SKILL.md` + `references/schema.md`). The other three carry compact inline copies because skills install standalone and can't share files.
- **`jarvis-search`** — read-only find (full-text + SIRA keyword match, default-on query expansion). References: `search-recipes.md`, `query-expansion.md`.
- **`jarvis-index`** — index/remove sources (SIRA keywords generated on index/re-index). References: `ingestion.md`, `sira-index.md`.
- **`jarvis-doctor`** — audit/fix (missing/un-indexed/stale/duplicate sources, stale keywords) with confirm gates + a Mermaid decision diagram. References: `diagnostics.md`.

All four are de-pluginized (no sibling-skill or MCP-tool hard deps).

**Maintenance rule:** the `jarvis` umbrella is the canonical index contract. When you change the model, update the inline copies in the three sub-skills to match.

**KB root:** `$JARVIS_KB` → `<git-root>/.jarvis/` when inside a git repo (`git rev-parse --show-toplevel`; auto-added to that repo's `.gitignore`) → `~/.jarvis/` → ask. The index is **not** stored in this repo.

**Runtime dependency:** `ripgrep` (`rg`) — every query/check is `rg` over the live index. No daemon, no database.

## Committing

This repo uses git on the `main` branch, with `origin` at `git@github.com:eoinhurrell/jarvis.git`. A `.gitignore` keeps local harness installs (`.claude/`, `.agents/`) out of the tree. Prefer conventional commits whose bodies explain the *benefit* of the change.
