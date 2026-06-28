# Jarvis Search Recipes (rg-only)

No persisted index. Every query is `rg` over `metadata/` (the generated markdown). Run from the index root. Scope to `metadata/` so you don't search the verbatim `sources/` originals. Metadata files always start with a YAML frontmatter block, so field queries anchor on `^field:`.

## Full-text

```bash
# basic, case-insensitive, with context and filename
rg -i -n -C1 'query terms' metadata/

# title of every hit (frontmatter title), useful for a result list
rg -l -i 'query terms' metadata/ | while read -r f; do
  printf '%s\t%s\n' "$(rg -m1 '^title:' "$f" | sed 's/^title:[[:space:]]*//')" "$f"
done

# rank by match density (most hits first)
rg -c -i 'query terms' metadata/ | sort -t: -k2 -rn
```

## Keyword-field matching (SIRA doc-side)

Each metadata file carries SIRA-generated `keywords:` — vocabulary a user might search with that *isn't* in the source text. Match a term against that field:
```bash
# sources indexed with a given keyword (token match, not substring)
rg -l '^keywords:.*\bk8s\b' metadata/
```
This is how a query for "k8s" finds a source whose text only says "Kubernetes".

## Source resolution

Every metadata file has a `source:` block pointing at its original.
```bash
# print the original's relative path for a metadata file
rg -m1 '^[[:space:]]*path:' -or '$1' metadata/<file>.md   # under the source: block
# all indexed sources of a given kind
rg -l '^[[:space:]]*kind:[[:space:]]*pdf' metadata/
```
To hand back the original, read `source.path` and present the file at `<kb-root>/<that-path>`.

## Weighted-density ranking (with query expansion)

When combining original query terms (`ORIG`, a `|`-joined alternation) with SIRA-expanded terms (`EXP`, post DF>0), score each hit by original-term hits plus a discounted count of expansion-term hits (`w` ≈ 0.5, tunable):
```bash
for f in $(rg -l -i -e "$ORIG" -e "$EXP" metadata/); do
  o=$(rg -c -i -e "$ORIG" "$f" 2>/dev/null || echo 0)
  e=$(rg -c -i -e "$EXP" "$f" 2>/dev/null || echo 0)
  echo "$(( o + e / 2 )) $f"      # w=0.5 → divide e by 2; tune the divisor
done | sort -rn
```
A source matching an original term outranks one matching only via expansion.

## Result presentation
Return: `title · id · path`, then the matching snippet (`rg -C1`). Don't dump full bodies. Always offer the resolved original via `source.path`.
