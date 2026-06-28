# Jarvis Search Recipes (rg-only)

No persisted index. Every query is `rg` over the live tree. Run all of these from the KB root. Markdown only: `--type md` (or `-g '*.md'`). Notes always start with a YAML frontmatter block, so field queries anchor on `^field:`.

## Full-text

```bash
# basic, case-insensitive, with context and filename
rg -i -n -C1 'query terms' -g '*.md'

# title of every hit (frontmatter title), useful for a result list
rg -l -i 'query terms' -g '*.md' | while read -r f; do
  printf '%s\t%s\n' "$(rg -m1 '^title:' "$f" | sed 's/^title:[[:space:]]*//')" "$f"
done

# rank by match density (most hits first)
rg -c -i 'query terms' -g '*.md' | sort -t: -k2 -rn
```

Exclude the `sources/` originals from text search if you ever store text-y originals there: add `-g '!sources/**'` (the derived notes carry the searchable text anyway).

## Frontmatter / structured queries

Fields: `id, title, type, status, project, team, owner, created, updated, tags, source`.

```bash
# all notes of a type
rg -l '^type:[[:space:]]*project' -g '*.md'

# by status
rg -l '^status:[[:space:]]*active' -g '*.md'

# by owner / team / project
rg -l '^team:[[:space:]]*retrieval' -g '*.md'

# tag membership (tags: [a, b, c]) — match the token, not a substring
rg -l '^tags:.*\bretrieval\b' -g '*.md'
```

### Multi-field (AND) — chain with `-l` then filter
`rg` is single-pattern for the AND case; intersect file lists:

```bash
# active projects owned by team "retrieval"
comm -12 \
  <(rg -l '^type:[[:space:]]*project'  -g '*.md' | sort) \
  <(rg -l '^status:[[:space:]]*active' -g '*.md' | sort) \
| xargs -r rg -l '^team:[[:space:]]*retrieval'

# accepted decisions tagged retrieval
comm -12 \
  <(rg -l '^type:[[:space:]]*decision'  -g '*.md' | sort) \
  <(rg -l '^status:[[:space:]]*accepted' -g '*.md' | sort) \
| xargs -r rg -l '^tags:.*\bretrieval\b'
```

### Rollups (counts by field)
```bash
# how many notes per type
rg -I -N '^type:[[:space:]]*(\S+)' -or '$1' -g '*.md' | sort | uniq -c | sort -rn

# status breakdown
rg -I -N '^status:[[:space:]]*(\S+)' -or '$1' -g '*.md' | sort | uniq -c | sort -rn
```

## Ingested-source resolution
A note with a `source:` block came from a non-markdown original.
```bash
# is this note an ingested doc? print the original's relative path
rg -m1 '^[[:space:]]*path:' -or '$1' note.md   # under the source: block
# all ingested notes of a given kind
rg -l '^[[:space:]]*kind:[[:space:]]*pdf' -g '*.md'
```
To hand back the original, read the note's `source.path` and present the file at `<kb-root>/<that-path>`.

## Result presentation
Return: `title · type/status · id · path`, then the matching snippet (`rg -C1`). Don't dump full bodies. If a hit has a `source:` block, also offer the original file.
