---
name: embeddings-and-vector-search
description: Engineering the vector-search layer itself — choosing embedding models (MTEB skepticism, dimensions, Matryoshka, late-interaction), ANN index selection (HNSW/IVF/flat/quantized/disk-based), pgvector-vs-dedicated-engine judgment, filtered-search recall failures, similarity-metric subtleties, embedding versioning, and recall@k evaluation. Load when picking an embedding model or vector store, sizing/tuning an index, debugging bad ANN recall, or planning a re-embed/migration.
---

# Embeddings & Vector Search Engineering

Scope: the vector layer — models, indexes, stores, metrics, operations. Pipeline-level RAG design lives in `rag-systems`.

## The standard doctrine, compressed

A strong model already produces this cold; anchors only. ANN degrades silently (never errors, returns worse neighbors) — measured recall@k vs exact search is an SLO, evaluated *with production filters applied*. Filtered HNSW craters on selective filters (traversal starves); pgvector ≥0.8 fixes it with `hnsw.iterative_scan = relaxed_order` + `hnsw.max_scan_tuples`; Qdrant with filterable-HNSW payload indexes (+`is_tenant` co-location, exact-search fallback below `full_scan_threshold`); and the universal move: an always-present categorical filter (tenant) is a **partition key, not a WHERE clause** — per-tenant partitions/namespaces turn the hardest filter case into no filter. <~100k vectors: no index, flat scan, recall 1.0. Scores are ordinal (anisotropy: unrelated pairs land 0.5–0.7); never ship `score > 0.75` — rank cutoffs or calibrated reranker scores. Matryoshka truncation → re-normalize or IP rankings skew. Binary quantization = 32×, recovered by 2–5× oversample + full-precision rescore, bad below ~512–768 dims. pgvector operator must match the index opclass (`<#>` = *negative* inner product; mismatch = silent seq scan — check EXPLAIN). MTEB = shortlist filter only (contaminated, gamed, averaged over irrelevant tasks); decide on ~100 labeled in-domain queries. Model upgrade = blue-green re-embed with dual-write; never a mixed-model index. HNSW: `m`=16, `ef_construction` 64–200, `ef_search` is the runtime dial (default 40 is a demo setting — sweep to the knee); churn tombstones rot recall until `REINDEX`/vacuum. Asymmetric models need their `query:`/`passage:`/instruction prefixes wired into one shared embed function. Zero/NaN vectors from failed embed calls poison ordering — validate at ingest.

## Sharpenings and the fast-moving layer

- **The landscape (as of 2026 — this table is the part that rots; verify before citing).** Multilingual: Qwen3-Embedding (open, 0.6B/4B/8B, top of MMTEB among open weights), Gemini embeddings, Cohere embed-v4 (also the main multimodal text+image API), BGE-M3 (workhorse open multilingual; dense+sparse+multi-vector from one model). Code: voyage-code models. Strong general APIs: voyage-3.5/voyage-3-large, Cohere v4, OpenAI text-embedding-3-large. Most 2026 models are Matryoshka-trained (voyage-3.5: 2048/1024/512/256; Cohere v4: 1536/1024/512/256; OpenAI `dimensions` param) — default to ≤1024 unless your eval shows the full dimension pays; OpenAI's own data had 3-large@256 beating ada-002@1536.
- **Late interaction is now production-real, with a specific home.** Qdrant has native multivector support (since 1.10), and **ColPali-class models are the standard approach for visual-document retrieval** (PDFs/slides as images, no OCR) as of 2026. Storage math before adopting: 300-token chunk × 128-d × fp16 ≈ 77 KB vs 2–4 KB single-vector — 20–40×; use as a reranking stage or for bounded high-value corpora, never the primary billion-scale index.
- **pgvector scale judgment (2026 rules of thumb).** Comfortable to ~5–10M vectors/table; 10–50M workable with `halfvec`, partitioning, or **pgvectorscale** (StreamingDiskANN + statistical binary quantization — check your managed Postgres offers the extension *before* promising it); beyond ~50M dedicated engines are the safer default. Default position stays: on Postgres already → pgvector until a *named* limit (scale, sustained-QPS-vs-OLTP contention, or a feature: Qdrant multivector, Milvus GPU/billion-scale, **Turbopuffer** object-storage namespaces — the cost floor for thousands of tenant namespaces, p99 in tens of ms). "Vector DBs are faster" in the abstract is not a reason; migrating buys a dual-write/backfill/cutover pipeline and loses transactional metadata joins.
- **Dual-write drift is your bug to own.** Vectors outside the source-of-truth DB = an eventual-consistency pipeline: outbox/CDC, DLQ, nightly reconciliation (source rows vs index points + content-hash spot checks). "We call both APIs in the request handler" is not a pipeline.
- **Long context windows are for not crashing, not for embedding well.** 8k–32k-window models still embed 200–500-token focused chunks better than 4k grab-bags; doc-level vectors of long docs leave the trained query–passage regime. Boilerplate across chunks makes cosine-near mean "shares header" — dedupe by content hash and near-identical-vector detection at ingest. Changing chunking changes the vector distribution: re-tune `ef_search`/`nprobe`, and IVF centroids especially assume the old distribution (never build IVF before representative data is loaded).
- **Negation/aggregation don't exist here.** "Docs NOT about X" (embeddings of X and not-X are neighbors), counts, most-recent — route to SQL/rerankers; don't tune the index hoping.

## Memory arithmetic first (decides the index; one line each)

```text
20M × 1024d: fp32 82 GB (+HNSW graph ~1.2–1.4× → 100–115 GB RAM) | halfvec 41 GB | int8 20 GB | binary 2.6 GB | MRL-512+binary 1.3 GB
→ full-precision HNSW = 128 GB box; halfvec = 64 GB; BQ-in-RAM + fp16-on-NVMe rescore = 16 GB box at ~0.95–0.98 of full recall (measure, don't assume).
```

## ANN-recall harness (isolates index loss from model loss — different bugs, different fixes)

```python
def ann_recall(queries, ann_search, X, ids, k=10):   # 200+ REAL queries, WITH production filters
    hits = tot = 0
    for q in queries:
        truth = set(ids[i] for i in np.argsort(-(X @ q))[:k])   # brute force, same vectors
        hits += len(truth & set(ann_search(q, k))); tot += k
    return hits / tot
# Sweep ef_search: 40→0.91  80→0.962  160→0.985  320→0.991(2.1× latency) — run at the knee.
```

ANN-vs-exact ≈0.99 but users can't find things → the index is exonerated; the loss is model/chunking/query handling. For *model* loss you need labeled query→doc pairs (mine logs, LLM-judge with ~10% human audit).

## Verification / self-check

- Arithmetic done (bytes, graph overhead, RAM/disk placement) for the actual N and D before choosing an index.
- Recall measured twice — ANN-vs-exact and labeled-query — both with production filters.
- Versioning enforced: model ID + dims + normalization + prefix convention pinned to the collection (put the model ID in the index *name*); mismatch is a startup error; a budgeted blue-green re-embed path exists.
- Operator matches opclass (EXPLAIN-verified); quantized indexes rescore with full precision; no absolute score thresholds anywhere.
- Model names/leaderboard standings marked "as of 2026" and re-verified rather than asserted.
- Stopping rule: filtered recall@k on target at budget latency with ≥30% memory headroom → stop tuning the index; remaining quality problems live upstream.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 12 baseline (cut/compressed — filtered-HNSW mechanics incl. exact pgvector GUCs and Qdrant is_tenant, partition-beats-filter, anisotropy, MRL re-norm, BQ oversample+rescore, opclass mismatch, flat-scan threshold, MTEB process, ColPali/late-interaction math, blue-green migration, HNSW params/tombstone rot, silent-failure inventory), 1 partial (model landscape naming — sharpened), 0 hard deltas.
- Biggest baseline gaps: none factual; Opus's weakest spots were current model/version specifics (voyage-3.5 vs voyage-3, Qdrant 1.10 multivector) and pgvector scale thresholds/pgvectorscale — the fast-rotting layer this file now concentrates on.
- Kept as likely-delta despite unprobed: dual-write reconciliation discipline, Turbopuffer namespace economics, chunking↔distribution retune interaction.
