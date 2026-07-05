---
name: rag-systems
description: Designing, building, and debugging retrieval-augmented generation — chunking by document structure, hybrid search, rerankers, metadata filtering, retrieval vs generation evaluation, context placement, query rewriting, and when RAG is the wrong tool. Load when building search-over-documents features, diagnosing bad RAG answers, or deciding between RAG, long-context, fine-tuning, or SQL.
---

# RAG Systems

## Core mental model (anchors — you know these; enforce them against shortcuts)

1. RAG quality is retrieval quality. Debug ingestion → chunking → retrieval → ranking → placement → generation, and *read the retrieved chunks* for the failing query before proposing anything. "Be accurate" prompt edits on a retrieval miss is the classic wasted week.
2. Chunk by document structure (heading sections with heading-path prepended, AST functions, Q+A pairs, table+caption), size as constraint not rule; bad chunking is unfixable downstream.
3. Hybrid BM25+dense fused with RRF is the default (complementary failure modes; RRF avoids the incomparable-score-scales trap); embeddings recall wide (top 50–150), cross-encoder reranks narrow (top 3–10).
4. Structured constraints ("2024", "Acme", ACL) are metadata pre-filters, not text to embed.
5. Ground by contract: answer-from-context-only, chunk-ID citations, explicit "not in the documents" out, citations verified mechanically.
6. Question after the documents, best chunks at the edges, cap k — top-20 raw routinely loses to top-5 reranked at 4× the cost.

## The corrections (gaps in the default expert playbook)

**Quantitative corpora are a distinct failure class.** Embeddings barely separate "increase by 5%" from "decrease by 15%" — sign, magnitude, and units live in tokens dense retrieval treats as near-noise. For finance, specs, dosing, pricing: weight the BM25/exact leg harder, rerank aggressively, and require the generator to *quote figures verbatim with citation* rather than paraphrase — paraphrased numbers drift even when retrieval was perfect. Standard RAG reviews check recall and faithfulness and never slice errors by "answer contains a number."

**Conflicting sources need an executable resolution rule.** Two retrieved chunks disagree (old vs new pricing); the generator picks arbitrarily or blends them into a fact appearing in neither. The fix is two-part and both parts get skipped: (a) a resolution rule in the prompt ("prefer latest effective_date; if dates absent and sources conflict, state the conflict"), and (b) surfacing doc dates/versions in the chunk headers so the rule is *executable*. A rule referencing metadata the model can't see is decoration. Upstream, expire/downweight superseded docs at ingestion — recency metadata you didn't capture can't be retrofitted cheaply.

**Retrieve-then-filter ACL fails by k-starvation, not just leakage.** Everyone knows to pre-filter permissions inside the query. The missed mechanism: post-retrieval filtering also *silently degrades* — all top-k get filtered for a low-permission user and they receive "no results" for content they're entitled to, because their accessible docs never made the candidate set. Pre-filtering fixes both the security bug and the recall bug; that's why it's non-negotiable even where leakage seems handled.

**Decouple retrieval unit from generation unit.** Embed small precise chunks; hand the generator the parent section (small-to-big / parent-document retrieval). For tables: embed a text *summary* of the table, store the raw table for the generator — embeddings of raw cells are near-noise. The reflex "one chunk store serves both roles" quietly caps either recall or answer quality.

**Latency budget goes to the wrong stage.** Typical shape: embed ~10ms, ANN 10–50ms, rerank 100 candidates 50–300ms, generation 2–10s. Teams cut the reranker "for speed" while the generator emits 800 unneeded tokens. The reranker is the best quality-per-millisecond in the stack; generation length is where the latency actually is.

**Measure the unanswerable set explicitly.** Include queries whose answer is absent from the corpus; require "not found in the documents." This subset is where user trust is won and it's absent from almost every eval that otherwise splits recall@k from faithfulness. (For the negative path to fire, demonstrate it — a rule alone under-triggers; see prompt-engineering.)

## Compressed checklist (one-line reminders; each is a real incident)

- Overlap 10–20% only where boundaries are unreliable (OCR, transcripts); overlap on clean structural chunks duplicates index entries.
- e5/BGE-family: query/passage prefixes in one shared function used by indexer and query path — omission is a silent multi-point recall loss.
- One canonical `normalize(text)` for both ingestion and query; version the index with the embedding-model ID and make mismatch a startup error.
- Dump and read 30 random chunks before tuning anything; dedupe near-identical docs and cap chunks-per-doc in top-k (or MMR).
- Similarity thresholds aren't calibrated across queries — rank-based selection + reranker-score gating.
- Query rewriting for conversational follow-ups is mandatory; testers ask standalone questions, users don't. Decompose multi-part questions; expansion dictionaries for acronym-heavy BM25.
- Incremental upsert/delete keyed on doc ID + content hash from day one; "rebuild weekly" fails at the first corrected price.
- Grid {chunk strategy} × {k} × {rerank} against a labeled query set (50–200 real queries, paraphrased not copied from docs); recall@k separately from faithfulness-given-context.
- <~100k chunks: exact search (pgvector/FAISS flat) — don't buy ANN complexity; at scale, measure ANN recall vs exact on the gold set.
- RAG is the wrong tool for: aggregation questions (→ SQL), context-sized corpora (→ stuff + prefix-cache), style/persona goals (→ fine-tune).

## Worked micro-example — RRF (the whole trick)

```python
def rrf(rankings, k=60):                      # rankings: lists of chunk_ids, best first
    scores = {}
    for r in rankings:
        for rank, cid in enumerate(r):
            scores[cid] = scores.get(cid, 0) + 1.0 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)

top = rerank(query, rrf([bm25_top(q,100), dense_top(q,100)])[:100])[:5]
```
No score normalization, no tuned weights — ranks only, which is why it's robust; k=60 rarely needs tuning. Interpret your ablation: dense≈hybrid → paraphrastic users; bm25≈hybrid → identifier-heavy corpus.

## Verification / self-check

- Read retrieved chunks for ≥5 failing queries before any fix; read 30 random index chunks for self-containment.
- Separate recall@k and faithfulness numbers on real-usage-style queries, *including the unanswerable subset* and a numeric-answer slice.
- Chunk headers carry doc date/version; prompt contains a conflict-resolution rule that references them.
- ACL as pre-filter (check for k-starvation with a low-permission test user); embedding-model version pinned; superseded docs expire.
- Question after documents; citations spot-checked mechanically; figures quoted verbatim on quantitative corpora.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 13 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - No awareness of the numbers/units failure class (embeddings blur sign/magnitude; quote-verbatim requirement for quantitative corpora).
  - Conflicting-source handling absent: resolution rule + dates in chunk headers so the rule is executable.
  - Pre-filter ACL known for security but not the k-starvation recall mechanism; small-to-big retrieval and rerank-vs-generation latency arithmetic not surfaced.
