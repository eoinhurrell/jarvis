# Jarvis Index — SIRA keyword generation

Jarvis search is `rg`/lexical: it finds what's *written*, so it misses when a user's vocabulary differs from the note's (a note says "Kubernetes"; a user searches "k8s"). **SIRA** (*SuperIntelligent Retrieval Agent*, arXiv 2605.06647) closes that gap by having an LLM add, to each document, search terms **not present in the text** — synonyms, abbreviations, alternate names, layperson terms. SIRA reports +30% Recall@10 over BM25 on BEIR, strongest on exactly this vocabulary-mismatch problem. Combined with query-side expansion (see `jarvis-search`), both sides of the gap are bridged.

**You run the prompt yourself** — you are the LLM. No external API, no helper script, no daemon (consistent with Jarvis's model).

## When to generate
- **Create** — after writing the note.
- **Ingest** — after writing the derived note, against its extracted text.
- **Update** — regenerate whenever the body substantively changes.
- Skip if the note is a stub with no meaningful text.

## The index-time prompt

Apply this to the note's text (title + body; for ingested notes, the extracted text). Substitute `{max_n}` with **3** by default (tunable — lower for terser KBs, higher for technical/long-form).

```
You are improving a search index. Given a document, generate NEW search terms that are NOT already present in the document text.

Document:
{doc_text}

Your task: identify what vocabulary a user might use to search for this document that is MISSING from the document itself — synonyms, abbreviations, alternate names, related concepts, or layperson terms for the technical concepts above.

Rules:
- Do NOT include any word or phrase (or close variant) already present in the document text above.
- Generate at most 10 phrases, 1-{max_n} words each.
- Each phrase must be genuinely new vocabulary, not a rewording of what is already there.
- Be DIVERSE: cover as many distinct concepts from the document as possible — do not generate multiple synonyms for the same idea.

Think shortly, not more than 10 sentences. Then output only:
{{"keywords": ["...", "..."]}}
```

## Storage contract

Parse the `{"keywords": [...]}` JSON and write two frontmatter fields:
```yaml
keywords: [k8s, container orchestration, pods]
index_generated: 2026-06-28
```
- `keywords` is a **top-level** field (not nested) so it is hit by both full-text `rg` and `^keywords:` structured queries.
- Normalize: lowercase, hyphenated, drop duplicates and any term that (case-insensitively) already appears in the note text — the prompt forbids them, but verify.
- ≤ 10 terms.

## Staleness
`index_generated:` records when keywords were last produced. On update, if `updated:` is newer than `index_generated:` and the body changed, regenerate and refresh `index_generated:` (keep `id`/`created`). Unchanged body → leave keywords as-is.

## Caveat: over-generation
Generic terms (e.g. "overview", "summary") add noise. The prompt's *diversity* and *not-already-present* rules push away from boilerplate; that's sufficient for v1. Full DF/IDF pruning (dropping terms common across the whole KB) is out of scope — `rg` can't compute IDF, and match-density ranking already down-weights non-discriminative terms at query time.
