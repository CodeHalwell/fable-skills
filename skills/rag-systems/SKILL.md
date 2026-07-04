---
name: rag-systems
description: Designing, building, and debugging retrieval-augmented generation — chunking by document structure, hybrid search, rerankers, metadata filtering, retrieval vs generation evaluation, context placement, query rewriting, and when RAG is the wrong tool. Load when building search-over-documents features, diagnosing bad RAG answers, or deciding between RAG, long-context, fine-tuning, or SQL.
---

# RAG Systems

## Core mental model

1. **RAG quality is retrieval quality; the generator can't cite what it never saw.** When a RAG answer is wrong, assume retrieval failed until you've *looked at the retrieved chunks* for that query. Most "hallucination" complaints in RAG systems are retrieval misses the generator papered over. Debug the pipeline in order: ingestion → chunking → retrieval → ranking → placement → generation. Fixing the last stage first is the classic wasted week.
2. **Chunking is a document-structure problem, not a token-count problem.** A fixed 512-token splitter cuts tables from their headers, code from its function signature, and answers from the questions above them. The right unit is the document's own semantic unit — section, subsection, table+caption, function, FAQ pair — with size as a *constraint*, not the *rule*. Bad chunking cannot be fixed downstream by any embedding model or reranker: the information needed to answer simply isn't co-located in any retrievable unit.
3. **Embeddings recall, rerankers rank — division of labor.** Bi-encoder embeddings compress each text into one vector independently; they're fast over millions of docs but blur specifics (negation, numbers, entity distinctions like "Java" vs "JavaScript"). Cross-encoder rerankers read query+passage *together* and are far more accurate but ~1000× more expensive per pair. Standard architecture: cast a wide cheap net (top 50–150 via embeddings+BM25), rerank to top 3–10. Trying to make embeddings alone do precision ranking — or running a reranker over the whole corpus — misassigns the jobs.
4. **Hybrid (BM25 + dense) is the default, not the upgrade.** Dense search misses exact identifiers, part numbers, error codes, rare names, and quoted phrases — exactly what users of internal-doc systems search for. BM25 nails those and whiffs on paraphrase; the failure modes are complementary. Fuse with Reciprocal Rank Fusion (RRF) — it needs no score normalization, which is the trap in weighted-sum fusion (BM25 and cosine scores live on incomparable scales). Start hybrid; drop a leg only with eval evidence.
5. **Structured constraints are filters, not similarity.** "2024 invoices from Acme" — the year and customer are metadata predicates to apply *before/within* vector search, not text to embed. Embedding structured constraints and hoping cosine similarity respects them is a category error; similarity happily returns a 2019 Acme invoice. This requires capturing metadata (date, source, product, version, access level) at ingestion — the step teams skip and then can't retrofit.
6. **The generator must be grounded by contract.** Instruct answer-from-context-only, require citations to chunk IDs, give an explicit "not in the provided documents" out, and *verify* citations mechanically post-hoc. Ungrounded RAG is just a chatbot with expensive decoration.

## Decision frameworks

**Is RAG even the right tool?**
| Signal | Use instead | Because |
|---|---|---|
| Questions are aggregations/filters over structured data ("average deal size in Q3", "how many tickets...") | Text-to-SQL / query engine over the real tables | Retrieval returns k passages; it cannot COUNT, SUM, or GROUP BY. RAG over exported table rows is the most common RAG misapplication |
| Corpus is small (fits comfortably in the model's context) and queried repeatedly | Whole corpus in context (+ prompt-prefix caching) | Zero retrieval failure modes; caching makes it cheap |
| You want the model to *speak differently* (style, format, domain jargon, persona) | Fine-tuning | Retrieval injects knowledge, not behavior; fine-tuning injects behavior, not (reliably) facts |
| Knowledge is stable, bounded, and must be internalized (e.g., a proprietary API the model must code against fluently) | Fine-tune *plus* docs-in-context | Fine-tuning alone hallucinates plausible-but-wrong details; context provides ground truth |
| Fresh/per-user/large/access-controlled knowledge, cited answers required | RAG | This is the actual RAG sweet spot: too big for context, too volatile for weights, needs per-user ACL filtering and citations |

**Chunking by document type:**
| Document type | Chunk unit | Notes |
|---|---|---|
| Markdown/HTML docs, wikis | Heading-bounded sections; split oversized sections at paragraph boundaries | Prepend the heading path ("Admin Guide > Backups > Restore") to each chunk — restores context the split removed and improves both embedding and generation |
| Code | Function/class via AST or tree-sitter | Include signature + docstring; never split mid-function |
| PDFs with tables | Table + caption as one chunk; consider a text summary of the table as the *embedded* text with the raw table stored for the generator | Embeddings of raw table cells are near-noise |
| FAQs, Q&A, chat logs | One Q+A pair per chunk | The Q is the best retrieval key for the A; never separate them |
| Contracts/legal | Clause/numbered-section | Cross-references argue for including parent-section context |
| Transcripts | Speaker-turn groups within topic segments | Fixed windows split answers from questions |
Add 10–20% overlap only where boundaries are unreliable (OCR'd PDFs, transcripts); overlap on clean structural chunks mostly duplicates index entries. Also decouple *retrieval unit* from *generation unit* when useful: embed small precise chunks, but hand the generator the parent section ("small-to-big" / parent-document retrieval).

**Query handling before retrieval:**
- Conversational follow-ups ("what about the enterprise tier?") → rewrite into a standalone query using chat history (cheap model call). Skipping this is the #1 cause of "RAG works in testing, fails in real chat" — testers ask standalone questions, users don't.
- Multi-part questions ("compare X's and Y's refund policies") → decompose into sub-queries, retrieve per sub-query, merge (dedupe by chunk ID, then rerank the union).
- Vague queries → HyDE-style expansion (generate a hypothetical answer, embed that) can help when query and document vocabularies diverge; measure before adopting, it adds latency and sometimes hurts on precise queries.
- Acronym/synonym-heavy domains → maintain an expansion dictionary applied to the BM25 leg.

**Placement in the prompt ("lost in the middle"):** models attend most reliably to the start and end of the context; relevance of mid-context passages degrades as context grows. So: put the strongest chunks first (or split best-first/second-best-last), put the *question after the documents*, restate the core instruction at the end, and cap k at what demonstrably helps — top-20 raw chunks routinely underperforms top-5 reranked, at 4× the cost. More context is not more recall once the answer's already in there; it's more places to get lost.

## Failure modes and pitfalls

- **Evaluating only end-to-end.** One "answer quality 7/10" number can't tell you whether retrieval missed or generation ignored. Split metrics: (a) *retrieval recall@k* — for a labeled set of (query → gold chunk/doc) pairs, is the gold in the top k? (b) *generation faithfulness* — given fixed retrieved context, is every claim supported by it? Build the labeled query set first (50–200 real queries; mine logs, include paraphrases and known-hard cases). If recall@10 is 60%, no prompt engineering will save you; if recall is 95% and answers are wrong, now it's a prompting/grounding problem.
- **Testing with queries copied from the documents.** Verbatim phrasing inflates both BM25 and embedding scores. Real users paraphrase, misspell, and use different vocabulary. Eval queries must come from real usage or be adversarially paraphrased.
- **Similarity-threshold cutoffs as relevance gates.** Cosine scores aren't calibrated across queries — 0.78 is great for one query, noise for another. Prefer rank-based selection + reranker score gating; if you must threshold raw similarity, calibrate per-corpus on labeled data and expect it to break when you change embedding models.
- **Changing the embedding model without re-embedding the entire corpus.** Query vectors from model B against document vectors from model A produce garbage that still *returns results* — nearest neighbors of nonsense are still neighbors. Version the index with the embedding model ID; make mismatch a startup error, not a silent degradation.
- **Ingestion garbage nobody looked at.** PDF extraction that interleaves two columns, headers/footers repeated into every chunk, OCR mojibake, boilerplate navigation text dominating the index. Before tuning anything: dump 30 random chunks and *read them*. If a human can't answer from the chunk, neither can the model. Also dedupe near-identical docs (old versions, mirrored pages) — they crowd out diverse results in top-k.
- **Stale index with no update path.** Docs change; deleted/superseded content keeps getting retrieved and confidently cited. Design incremental upsert/delete keyed on stable doc IDs + content hash from day one; "we'll rebuild weekly" fails the first time someone corrects a wrong price.
- **Access control applied after retrieval — or never.** Retrieve-then-filter leaks via k starvation (all top-k get filtered, user gets nothing) and via the generator having seen forbidden text if filtering is buggy. Apply ACL as a metadata *pre-filter* inside the vector/BM25 query. A RAG system over mixed-permission docs without this is an incident, not a feature.
- **Reranker fed too little or too much.** Reranking top-5 embeddings output just reorders the same misses (recall is already lost); reranking 1000 blows the latency budget. Feed the reranker the fused top 50–150.
- **Citations that don't verify.** The model cites [3] but the claim comes from nowhere. Post-hoc check: for each cited claim, verify overlap/entailment against the cited chunk (string containment for quotes/numbers; a cheap NLI or judge call for paraphrases); strip or flag unverifiable claims. Also enforce the negative path in evals: queries whose answer is absent from the corpus must yield "not found in the documents" — measure this *unanswerable set* explicitly; it's where trust is won or lost.
- **Ignoring the numbers/units failure class.** Embeddings barely distinguish "increase by 5%" from "decrease by 15%". For quantitative corpora (finance, specs, dosing), lean harder on BM25/exact match and reranking, and require the generator to quote figures verbatim with citation rather than paraphrase them.
- **k and chunk-size tuned by anecdote.** These interact (bigger chunks → smaller k budget) and are corpus-dependent. Grid over {chunk strategy} × {k} × {rerank on/off} against the labeled query set; anecdotal tuning reliably picks a local optimum from the last demo query someone ran.

## Worked micro-examples

**1. Diagnosing "the bot said our refund window is 30 days; policy doc says 14."**
```text
Step 1: pull the logged retrieved chunks for that query.
Finding: top-5 contains a 2022 blog post ("30-day holiday returns") at rank 1;
the current policy doc chunk is rank 23.
Diagnosis chain: (a) stale content not deduped/expired at ingestion,
(b) no recency/authority metadata to filter or boost,
(c) no reranker — blog phrasing happened to sit closer in embedding space.
Fixes in order of leverage: expire/downweight superseded docs at ingestion;
add doc_type + effective_date metadata, filter policy questions to doc_type=policy;
add reranker over top-100. NOT a fix: telling the generator "be accurate."
```
The pattern: the generator faithfully reported what retrieval fed it. The bug was three stages upstream.

**2. Hybrid retrieval with RRF (the whole trick, runnably small):**
```python
def rrf(rankings: list[list[str]], k: int = 60) -> list[str]:
    # rankings: each is a list of chunk_ids, best first (e.g. [bm25_ids, dense_ids])
    scores = {}
    for ranking in rankings:
        for rank, cid in enumerate(ranking):
            scores[cid] = scores.get(cid, 0) + 1.0 / (k + rank + 1)
    return sorted(scores, key=scores.get, reverse=True)

candidates = rrf([bm25_top(query, 100), dense_top(embed(query), 100)])[:100]
top = rerank(query, candidates)[:5]        # cross-encoder, e.g. a Cohere/BGE reranker
```
Note what's absent: no score normalization, no tuned fusion weights — RRF uses only ranks, which is why it's robust across corpora. `k=60` is a standard default; it rarely needs tuning.

**3. Retrieval eval harness (build this before tuning anything):**
```python
# gold.jsonl: {"query": "...", "gold_chunk_ids": ["doc7#s3", ...]}  ~100 real queries
def recall_at_k(retriever, gold, k=10):
    hits = sum(bool(set(retriever(g["query"], k)) & set(g["gold_chunk_ids"])) for g in gold)
    return hits / len(gold)

for name, r in {"bm25": bm25_only, "dense": dense_only,
                "hybrid": hybrid, "hybrid+rerank": hybrid_rerank}.items():
    print(name, recall_at_k(r, gold, 5), recall_at_k(r, gold, 20))
```
Typical shape of results on real corpora: hybrid beats either leg alone; rerank lifts recall@5 toward recall@20 of the un-reranked candidates. If dense alone ≈ hybrid, your users' queries are unusually paraphrastic; if bm25 alone ≈ hybrid, your corpus is identifier-heavy — either way you now *know*, per-corpus, instead of guessing.

## Verification / self-check

- Have you read actual retrieved chunks for at least 5 failing queries before proposing any fix? (If not, you're guessing.)
- Read 30 random chunks from the index: is each self-contained enough for a human to answer from? Headings attached, tables intact, no boilerplate?
- Do you have separate numbers for retrieval recall@k and generation faithfulness, measured on real-usage-style queries including an unanswerable subset?
- Index hygiene: embedding-model version pinned to the index, incremental update path exists, superseded docs expire, ACL applied as pre-filter.
- Prompt placement: question after documents, best chunks at the edges, explicit "not in the documents" escape hatch, citations mechanically spot-checked.
- Before recommending RAG at all: confirm the questions aren't aggregation-shaped (→ SQL), the corpus isn't context-sized (→ stuff it), and the goal isn't style transfer (→ fine-tune).
