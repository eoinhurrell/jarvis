# Jarvis Search — SIRA query expansion

At search time, expand the user's query with vocabulary a relevant answer note would contain but the query omits. This is the query-side half of **SIRA** (arXiv 2605.06647); the doc-side half is the `keywords:` frontmatter that `jarvis-index` writes. SIRA retrieval matches expanded query terms over original-text ∪ doc-keywords and scores `BM25(q_orig) + w·BM25(q_exp)` — in Jarvis that's `rg` match density with a weight on expansion hits.

**You run the prompt yourself** — you are the LLM. No external call.

## When to expand (default-on)
- **Expand** natural-language queries ("how do we handle on-call rotations", "notes about k8s scaling").
- **Skip** when the query is already precise: a note `id` (`YYYYMMDDThhmmss`), an exact quoted phrase, a filename/path, or a single known term the user wants verbatim. Also skip if the user says "exact" or otherwise opts out.

## The query-time prompt

Apply to the user's query. Substitute `{max_n}` with **3** by default (tunable).

```
You are expanding a search query to improve retrieval. Generate NEW terms that are NOT already present in the query.

Query:
{doc_text}

Your task: identify vocabulary that a correct answer document would contain but that is MISSING from the query — synonyms, domain terms, abbreviations, related concepts, or alternate phrasings the answer document would use.

Rules:
- Do NOT include any word or phrase already present in the query text above.
- Generate at most 10 phrases, 1-{max_n} words each.
- Each phrase must add new vocabulary not already in the query.
- Be DIVERSE: cover multiple distinct facets of what a correct answer would address — do not generate synonyms for the same concept.

Think shortly, not more than 10 sentences. Then output only:
{{"keywords": ["...", "..."]}}
```

## DF>0 filter (drop dead terms)
SIRA requires every expansion term to actually exist in the index (DF > 0). Before searching, drop any expanded term that matches **zero** notes:
```bash
# keep only expanded terms that hit at least one note (run from KB root)
for t in k8s "container orchestration" pods; do
  rg -ql -i -- "$t" -g '*.md' && echo "$t"   # echoed terms survive the filter
done
```

## Weighted-density ranking
Build the term sets: `ORIG_TERMS` = significant words from the original query; `EXP_TERMS` = expanded terms that survived DF>0 (each a `|`-joined alternation). Score each matching note by original-term hits plus a discounted count of expansion-term hits (`w` ≈ **0.5**, tunable — lower if expansion is noisy):
```bash
# from KB root
for f in $(rg -l -i -e "$ORIG_TERMS" -e "$EXP_TERMS" -g '*.md'); do
  o=$(rg -c -i -e "$ORIG_TERMS" "$f" 2>/dev/null || echo 0)
  e=$(rg -c -i -e "$EXP_TERMS" "$f" 2>/dev/null || echo 0)
  echo "$(( o + e / 2 )) $f"      # w=0.5 → divide e by 2; change divisor to tune
done | sort -rn
```
A note matching an original term outranks one matching only via expansion, mirroring SIRA's `w < 1`. Notes whose `keywords:` contain an expanded term get a hit here — that's the doc-side expansion paying off.

## Output
List results as `title · type/status · id · path`, then the snippet. Note briefly which terms (original vs expanded) drove each top hit so the user can see the expansion's effect.

## Caveats
- Over-generation adds false positives; DF>0 + the diversity rule + weighted ranking contain it.
- One generation step of latency per search — that's why we skip precise lookups.
- Expansion quality depends on the model knowing the domain (a SIRA limitation); for a corporate KB this is usually fine.
