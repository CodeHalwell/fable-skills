---
name: embeddings-and-vector-search
description: Engineering the vector-search layer itself — choosing embedding models (MTEB skepticism, dimensions, Matryoshka, late-interaction), ANN index selection (HNSW/IVF/flat/quantized/disk-based), pgvector-vs-dedicated-engine judgment, filtered-search recall failures, similarity-metric subtleties, embedding versioning, and recall@k evaluation. Load when picking an embedding model or vector store, sizing/tuning an index, debugging bad ANN recall, or planning a re-embed/migration.
---

# Embeddings & Vector Search Engineering

Scope: the vector layer — models, indexes, stores, metrics, operations. For pipeline-level RAG design (chunk strategy per document type, rerankers, prompt placement), load `rag-systems`; this skill covers what sits underneath it.

## Core mental model

1. **ANN is a lie you negotiate with.** Every non-flat index returns approximately the nearest neighbors, and "approximately" is a tunable you own. The recall you get is a function of index parameters, filters, quantization, and data distribution — and it degrades silently: an ANN query never errors, it just returns worse neighbors. Treat measured recall@k against exact search as a first-class SLO, not a benchmark you ran once.
2. **The whole game is a quadrilemma: recall × latency × memory × build/update cost.** Flat search maxes recall and update cost is zero, but latency grows linearly. HNSW buys sub-linear latency at high memory and slow builds. IVF is cheap to build and store but needs tuning and repartitioning. Quantization trades recall for memory, buyable back with oversampling+rescore. Disk-based (DiskANN-class) trades latency for memory. Every index decision is picking which corner to sacrifice — say which one out loud.
3. **Embedding vectors are opaque model artifacts, not portable data.** Vectors from different models — or different versions, or different truncation/normalization of the same model — are mutually meaningless, yet nearest-neighbor search over mixed vectors still returns confident results. Version the embedding (model ID + dims + normalization + any query/passage prefix) as part of the index's schema, and make mismatch a hard startup error.
4. **Similarity scores are ordinal, not cardinal.** Embedding spaces are anisotropic — vectors occupy a narrow cone, so cosine similarities of *unrelated* texts often land at 0.5–0.7 and the useful signal lives in a compressed band near the top. Ranks are trustworthy; absolute score thresholds are not, and they change every time you change models. Never ship a hardcoded `score > 0.75` relevance gate.
5. **Benchmarks (MTEB) are directionally useful and specifically gamed.** Frontier embedding models train on MTEB-adjacent data; leaderboard rank compresses dozens of tasks you don't care about into one number. As of 2026 the leaderboard's top spots churn monthly and gaps of <2 points are noise. Use MTEB to build a shortlist of 3–4 candidates; pick the winner on ~100 of *your* labeled queries.
6. **Filters and ANN fight each other.** Graph and cluster indexes are built over the whole corpus geometry; a metadata filter removes nodes mid-traversal, starving the search and cratering recall exactly on selective filters. "Add a WHERE clause" is the single most common way production vector search silently breaks — know how your engine handles it before you ship filtered queries.

## Choosing an embedding model — the reasoning chain

Ask in this order:

1. **What languages and modalities?** Multilingual or code or image+text prunes the list immediately. As of 2026: Qwen3-Embedding (open-weight, 0.6B/4B/8B) and Gemini embedding models lead multilingual; Cohere embed-v4 is the main multimodal (text+image) API option; voyage-code models for code. BGE-M3 remains the workhorse open-weight multilingual choice with dense+sparse+multi-vector outputs from one model.
2. **API or self-hosted?** Self-hosting an 8B embedding model needs a GPU and serving stack; the API price war has made hosted embeddings very cheap. Self-host when: data can't leave, throughput is huge and sustained, or you need to pin weights forever (see versioning). Otherwise API. Strong API options as of 2026: voyage-3.5 / voyage-3-large, Cohere embed-v4, OpenAI text-embedding-3-large, Gemini embeddings. Verify current names/pricing — this list rots fast.
3. **What dimension can you afford?** Dimension drives memory, index size, and latency linearly forever after. Most 2026 models are Matryoshka-trained (MRL): the first k dims are a usable embedding (voyage-3.5: 2048/1024/512/256; Cohere v4: 1536/1024/512/256; OpenAI via the `dimensions` param; Qwen3, Gemini, EmbeddingGemma natively). Default to 1024 or below unless eval shows the full dimension pays for itself — OpenAI's own data showed text-embedding-3-large truncated to 256 beating ada-002 at 1536. Truncated MRL vectors **must be re-normalized** after truncation.
4. **Asymmetric or symmetric?** Retrieval models mostly want distinct query/document treatment — either an `input_type` API param (Cohere, Voyage) or literal text prefixes/instructions (`query:`/`passage:` for e5-family; instruction strings for BGE/Qwen3/GTE). Check the model card; wire the convention into one shared embed function.
5. **Do you need late-interaction?** ColBERT-class models keep one small vector *per token* and score via MaxSim — much better at exact-term and fine-grained matching than single vectors, at ~50–100× storage (num_tokens × per-token dim). As of 2026 this is production-real: Qdrant has native multivector support (since 1.10), and ColPali-style models make late interaction the standard approach for *visual document* retrieval (PDFs as images). Reasoning: use single-vector for first-stage recall at scale; use late-interaction as a reranking stage or when the corpus is small/high-value enough to eat the storage. Don't build your primary billion-scale index on token-level vectors.
6. **Then eval on your data.** ~100 labeled (query → relevant doc) pairs, recall@10 per candidate model. Two hours of work; routinely reorders the MTEB shortlist, especially on jargon-heavy or non-English corpora.

## Choosing the index — the quadrilemma chain

1. **< ~100k vectors (or p99 budget is generous): no index.** Flat/exact scan. 100k × 1024d float32 is 0.4 GB; brute force is a few ms with SIMD. Recall is 1.0 by construction and there is nothing to tune, rebuild, or monitor. Buying ANN complexity here is negative-value work.
2. **In-RAM scale, read-heavy, recall matters: HNSW.** The default for 100k–~50M vectors. Graph traversal gives log-ish latency and 0.95–0.99 recall at sane settings. Costs: the graph lives in RAM alongside vectors (~1.2–1.5× raw vector size total for typical `m`), builds are slow and CPU-hungry, and deletes only tombstone (recall decays until rebuild/vacuum). Parameters that matter: `m` (16 default; graph degree — memory and build time), `ef_construction` (64–200; build-time quality), `ef_search` (the *runtime* recall/latency dial — tune this first, per query if the engine allows).
3. **Memory-constrained or rebuild-often: IVF.** Partition into `nlist` clusters, probe `nprobe` at query time. Near-zero memory overhead beyond vectors, builds ~10× faster than HNSW — but recall is bumpier (correct neighbors sit in unprobed cells, especially for out-of-distribution queries), and the centroids go stale as data drifts, requiring retraining. Starting points (per pgvector guidance): `lists ≈ rows/1000` (≤1M rows) or `sqrt(rows)` above; `probes ≈ sqrt(lists)`. IVF wants representative data *before* index creation — building it on an empty table then bulk-loading gives garbage centroids.
4. **Vectors ≫ RAM: quantize before you shard.** Order of escalation: (a) halve with fp16/halfvec — near-free recall-wise; (b) int8 scalar quantization — 4×, minor loss; (c) binary quantization — 32×, meaningful loss that you buy back by oversampling the binary index (fetch 2–5× k) and rescoring candidates with full-precision vectors (kept on disk). BQ works well on high-dim MRL-style embeddings (≥1024d) and badly on low-dim ones. Published example: ~0.98 recall@50 with just 2× oversampling on 1024d Cohere embeddings. Product quantization (PQ) compresses hardest but degrades most and is mainly an ingredient inside DiskANN-class systems now, not something you pick directly.
5. **Vectors ≫ RAM even quantized: disk-based graph (DiskANN class).** Quantized vectors in RAM for traversal, full vectors + graph on NVMe, ~1 disk read per hop. Implementations as of 2026: Microsoft DiskANN, Timescale's pgvectorscale (StreamingDiskANN + statistical binary quantization, keeps Postgres viable to ~50M+), Milvus DiskANN mode, Turbopuffer's object-storage-native design (S3-resident, RAM-cached hot set — the cost-floor option for huge, many-namespace, latency-tolerant workloads). Expect p99 in the tens of ms, not single digits.
6. **Update pattern check, always:** heavy delete/update churn punishes HNSW (tombstones) and IVF (centroid drift) differently. If the corpus turns over weekly, prefer an engine with real segment merging/vacuum (Qdrant, Milvus, Lucene-based) or plan periodic reindex as a scheduled job, not an emergency.

## pgvector vs dedicated engine — the judgment

Default position: **if the app already runs on Postgres, start with pgvector and keep it until a named limit is hit.** The reasoning: vectors next to their metadata rows give you transactional consistency (no dual-write drift between "real DB" and vector DB), SQL joins/filters for free, one backup/HA/access-control story, and zero new infra. As of 2026 pgvector (0.8.x) has HNSW with parallel builds, `halfvec` (fp16), binary vectors via the `bit` type, and — critically — iterative index scans (`hnsw.iterative_scan = relaxed_order`) that fix the classic filtered-query recall hole.

Named limits that justify a dedicated engine (Qdrant, Milvus, Weaviate, Turbopuffer, LanceDB, Vespa/Elastic):

- **Scale:** pgvector is comfortable to ~5–10M vectors per table on a well-provisioned instance; 10–50M is workable with care (halfvec, partitioning, or pgvectorscale); beyond ~50M, dedicated engines are the safer default. These are 2026 rules of thumb — the real constraint is "does the index fit in RAM you're willing to buy for your Postgres box, and does index build time fit your ops tolerance."
- **QPS:** vector search competes with your OLTP workload for the same buffer cache and CPU. Sustained thousands of vector QPS deserves its own tier (which can still be a Postgres read replica before it's a new database).
- **Feature need:** native multivector/late-interaction (Qdrant), built-in hybrid with sparse vectors, GPU indexing at billion scale (Milvus), object-storage economics for thousands of tenant namespaces (Turbopuffer), embedded/in-process serverless (LanceDB).
- **What is NOT a reason:** "vector DBs are faster" in the abstract, a benchmark blog, or resume-driven architecture. Migrating means building the dual-write/backfill/cutover machinery and losing transactional metadata joins — price that in.

## The filtered-search problem

Why it breaks: HNSW greedy search walks the graph toward the query; with `WHERE tenant_id = 42` matching 0.5% of nodes, almost every step lands on a filtered-out node. **Post-filtering** (search top-k, then filter) returns k×selectivity results — often zero. Naive **pre-filtering** (filter first, brute-force the survivors) is correct but only fast when the filter is very selective. Engines resolve this differently — know which one yours does:

- **pgvector ≥0.8:** iterative scans — keep walking the graph until enough filtered rows are found or a scan budget (`hnsw.max_scan_tuples`) is hit. Works; latency grows with filter selectivity. For a hard, always-present filter (tenant), consider partial indexes or partitioning by that key instead.
- **Qdrant:** filterable HNSW — extra graph links built using payload-index info, so traversal survives selective filters; plus a query planner that flips to exact search when the filtered set is small.
- **Weaviate/Milvus and most others:** bitmap pre-filtering fed into the ANN search (candidate-set intersection during traversal).
- **Universal expert move:** if one categorical filter appears in ~every query (tenant, language, product line), don't filter — **partition**: one index/collection/namespace per value. Turns the hardest filter case into no filter at all; this is Turbopuffer's namespace model and multi-tenant SaaS's correct default.
- Always eval recall@k *with production-realistic filters applied*. Unfiltered recall is the number everyone measures and nobody experiences.

## Metrics, normalization, anisotropy

- Most modern embedding APIs emit unit-normalized vectors; on unit vectors, cosine, dot product, and L2 give **identical rankings** (cosine = dot; L2² = 2−2·dot). Pick dot/inner-product ops when normalized — it skips the norm division (in pgvector: `vector_ip_ops`, and note `<#>` returns the *negative* inner product because operators must sort ascending).
- If you truncate Matryoshka dims, quantize, or average/pool vectors yourself, the result is **no longer unit-norm** — re-normalize before using inner product, or use cosine ops which tolerate it.
- Dot product on deliberately *unnormalized* vectors lets magnitude encode importance/popularity — some recommender embeddings exploit this; retrieval embeddings almost never do. Don't mix conventions in one index.
- Anisotropy (practical form): all-pairs similarity in a corpus might span 0.55–0.85. Consequences: absolute thresholds don't transfer across queries, corpora, or models; "similarity 0.8" is not evidence of relevance; score gaps *within one result list* are meaningful, raw values are not. If you must gate, gate on a reranker's calibrated score or on rank.

## Chunk–embedding interaction

- A single vector is a lossy summary; the longer and more multi-topic the input, the more it averages toward mush. Models advertising 8k–32k token windows still embed 200–500-token focused chunks better than 4k-token grab-bags for retrieval purposes. Long context windows are for *not crashing*, not for *embedding well*.
- Query–document asymmetry: 5-word queries against 400-token chunks is the regime these models are trained for. Embedding whole documents to "compare documents" or clustering on doc-level vectors of very long docs quietly leaves this regime — expect degraded geometry.
- Boilerplate poisons neighborhoods: headers/footers/nav repeated across chunks makes cosine-near mean "shares boilerplate," not "shares meaning." This manifests *in the vector layer* as clusters of near-duplicate vectors — dedupe by content hash and by near-identical-vector detection at ingest.
- When you change chunking you change the vectors' statistical distribution — re-tune `ef_search`/`nprobe` and re-measure recall; IVF centroids especially assume the old distribution.

## How an expert thinks through this

Scenario: multi-tenant SaaS doc search. 30M chunks across 4,000 tenants, on RDS Postgres today, p95 budget 150ms, filters on tenant + doc_type, embedding model not yet chosen.

*"First the model, because dimension drives every downstream cost. Multilingual? Yes, EU customers — so shortlist as of 2026: Cohere embed-v4, voyage-3.5, Qwen3-Embedding self-hosted, BGE-M3. Do I want 8B self-hosted? No — no GPU ops team, throughput is spiky; API it is. Dimension: 30M × 3072d float32 would be 369 GB of raw vectors — absurd. All shortlisted models are Matryoshka; I'll evaluate at 1024 and 512 on our labeled queries and expect 1024 to win by a hair and 512 to be good enough. Take 1024 pending eval.*

*Now the store. Reject 'move to a dedicated vector DB' as the opening move — metadata lives in these Postgres rows, tenant ACLs are already SQL, and dual-writing two databases is where consistency bugs breed. But check pgvector feasibility honestly: 30M × 1024d float32 = 123 GB raw, HNSW total maybe 160+ GB RAM — that's a monster instance competing with OLTP. Reject vanilla HNSW-in-Postgres at full precision.*

*Options: (a) halfvec HNSW → ~61 GB vectors, still big; (b) pgvectorscale/DiskANN → fits, adds an extension dependency my managed Postgres may not offer — check that before promising it; (c) binary quantization + rescore → 3.8 GB index, full vectors on disk for rescoring; (d) dedicated engine. Before picking, exploit the workload's shape: every query is single-tenant. 30M/4,000 ≈ 7.5k chunks per tenant median. That's the real unlock — partition by tenant and each search is brute-force over a few thousand vectors. No ANN index at all for most tenants, recall 1.0, no filtered-recall problem, no tuning. Keep an HNSW path only for the ten whale tenants with 1M+ chunks. Reject the fancy global index because the filter structure makes it unnecessary — this is the 'partition beats filter' move.*

*Whale tenants: HNSW per whale, halfvec, m=16, ef_construction=100, tune ef_search against a labeled set with the doc_type filter applied, iterative_scan=relaxed_order on. Eval harness before launch: 100 real queries per segment, recall@10 vs exact scan, filtered. Stopping rule: recall@10 ≥ 0.95 at p95 ≤ 150ms — hit that and stop tuning; chunking and reranking upstream now matter more than another recall point.*

*What would change my mind: chunk count 10×-ing (→ pgvectorscale or Turbopuffer namespaces — its per-tenant model maps 1:1 to this workload); a cross-tenant search feature (→ real global index, rethink); sustained QPS crushing the primary (→ replica first, engine second)."*

## Failure modes & pitfalls

- **Mixed-model vectors in one index.** Query embedded with model B against documents from model A returns plausible-looking garbage — no error, just quietly wrong neighbors. Also true for same model at different truncation dims, or normalized-vs-not. Fix: store `embedding_model`, `dims`, `normalized`, `prefix_convention` in the collection metadata; assert at query time; put the model ID in the index/collection *name* so a mismatch is impossible to miss.
- **Model upgrade without a migration plan.** You cannot upgrade embeddings in place — every vector must be regenerated. Correct pattern: blue-green — build index B with the new model (backfill can take days at API rate limits; budget it), dual-write during backfill, cut queries over atomically, drop A. Never route queries to a half-migrated index; a corpus that is 60% model-A and 40% model-B vectors ranks model-B docs incomparably against model-A docs.
- **Missing query/passage prefixes on asymmetric models.** e5/BGE/GTE/Qwen3-family expect `query:`-style prefixes or instruction strings; omitting them (or applying the query prefix at ingest) costs large recall with zero errors. One shared `embed(text, kind)` function used by indexer and query path.
- **MRL truncation without re-normalization.** Slicing `vec[:512]` from a unit 2048-d vector leaves norm <1; feed that to inner-product search and rankings skew. Truncate → re-normalize → then index. Same trap after any pooling/averaging.
- **Hardcoded similarity thresholds.** `if score > 0.75: relevant` breaks across queries today and completely on the next model. Anisotropy means the whole score distribution shifts. Use rank cutoffs, reranker scores, or per-corpus calibrated gates you re-fit on model change.
- **Filtered queries evaluated unfiltered.** Recall@10 = 0.97 in the notebook; production queries all carry `tenant_id` and effective recall is 0.4 with pre-0.8-pgvector-style post-filtering (or with `hnsw.iterative_scan=off`). Eval with the real WHERE clauses. In pgvector check `EXPLAIN ANALYZE` — if you see the index scan returning few rows then a Filter node discarding most, you're in the hole.
- **pgvector: forgetting the operator/opclass must match.** Index built with `vector_cosine_ops` is *unused* by a query ordering with `<#>` (inner product) — you get a silent sequential scan, correct results, terrible latency; or worse, an index scan with the wrong metric via a mismatched opclass. Match the query operator (`<=>` cosine, `<#>` neg-inner-product, `<->` L2) to the index opclass, and confirm the index is used with EXPLAIN.
- **IVF built before the data arrived.** `CREATE INDEX ... USING ivfflat` on an empty/small table trains centroids on nothing; recall craters after bulk load. Build IVF after loading representative data; rebuild after major distribution shifts. (HNSW doesn't have this trap, but bulk-load-then-index is far faster than index-then-load for both.)
- **HNSW delete rot.** Heavy churn leaves tombstoned nodes degrading traversal; recall sags over weeks — looks like "the model got worse." Monitor recall over time, not just at launch; schedule vacuum/optimize/rebuild per your engine's story (pgvector: `REINDEX CONCURRENTLY`; Qdrant/Milvus: segment optimization).
- **Binary quantization without rescoring.** BQ alone loses real recall; the design is BQ-search → oversample 2–5× → rescore with full-precision vectors. Skipping rescore (or not keeping full vectors around to rescore with) ships the loss. Also: BQ on 384-d embeddings is much worse than on 1024-d+; check dimension before adopting.
- **Buying HNSW's RAM bill with your eyes closed.** Estimate before building: vectors + graph must be resident or latency explodes 10–100× on page faults. The calc takes one line (below); do it before choosing, not after the OOM.
- **ef_search/nprobe left at defaults forever.** These are *the* runtime recall dials. Defaults (pgvector `hnsw.ef_search=40`) are tuned for demos. Sweep them against your labeled set: the recall/latency curve typically has an obvious knee; run at the knee.
- **Treating MTEB rank as the decision.** Adjacent leaderboard models differ by noise; contamination is documented and endemic; your corpus's jargon/language/length profile isn't MTEB's. The 2-point MTEB gap you're paying 3× more for routinely vanishes on in-domain eval. Shortlist by MTEB, decide by your own recall@k.
- **Late-interaction storage surprise.** ColBERT-class stores ~one vector per token: a 300-token chunk at 128-d/token fp16 ≈ 77 KB vs 2 KB for a single 1024-d fp32 vector — ~38× before compression. Adopting it corpus-wide without doing this multiplication is how a 100 GB index becomes 4 TB. Use as reranker over first-stage candidates, or for bounded high-value corpora (and for visual-document retrieval where ColPali-class is as of 2026 the standard).
- **Dual-write drift between source DB and vector store.** Doc updated in Postgres, embedding update job failed silently; search now returns stale text or dangling IDs. If vectors live outside the source-of-truth DB, you own an eventual-consistency pipeline: outbox pattern or CDC, dead-letter queue, and a nightly reconciliation count (`source rows` vs `index points`, plus content-hash spot checks). "We call both APIs in the request handler" is not a pipeline.
- **Assuming vector search does aggregation or negation.** "docs NOT about X", "the most recent doc about Y", counts, sorts by anything but similarity — the index answers none of these. Negation especially: embeddings of "X" and "not X" are near-neighbors. Route these to SQL/filters/rerankers; don't tune the index hoping.
- **Zero-vector and NaN ingestion.** Failed embedding calls that return zeros/NaNs get indexed happily; zero vectors have undefined cosine similarity (division by zero — pgvector returns NaN, which then poisons ordering). Validate `norm > 0` and finiteness at ingest; reject, don't default.
- **Evaluating ANN recall against vibes instead of exact search.** The only ground truth for *index* recall is brute-force on the same vectors: sample 200–1000 real queries, compute exact top-k (flat scan of the same table/collection), measure overlap with ANN top-k. This isolates index loss from model quality — different bugs, different fixes. For *model* quality you need labeled query→doc pairs; build them cheaply by mining logs and having a strong LLM judge candidate pairs (with ~10% human audit of the judgments).

## Worked micro-examples

**1. Memory sizing decides the index — do this arithmetic first.**
```text
Corpus: 20M chunks, 1024-d embeddings.

float32 raw:     20e6 × 1024 × 4 B  = 81.9 GB
+ HNSW graph (m=16 ≈ +80–150 B/vector + overhead): plan ~1.2–1.4× → ~100–115 GB RAM
halfvec (fp16):  20e6 × 1024 × 2 B  = 41.0 GB   (→ ~55 GB with graph)
int8 SQ:         20e6 × 1024 × 1 B  = 20.5 GB
binary (1 bit):  20e6 × 1024 / 8 B  =  2.6 GB   (fits in RAM on anything)
MRL truncate to 512d, then binary:     1.3 GB

Implications: full-precision HNSW needs a ~128 GB box — possible, expensive.
halfvec HNSW fits a 64 GB box. BQ index in RAM + fp16 vectors on NVMe for
rescoring (41 GB of cheap disk) fits a 16 GB box: search BQ with 3× oversampling
(fetch 30 for k=10), rescore 30 with exact distance. Expect ~0.95–0.98 of
full recall for ~1/30th the memory — measure it on your data, don't assume.
```

**2. pgvector (≥0.8, as of 2026) production shape — halfvec HNSW + filtered search done right:**
```sql
CREATE TABLE chunks (
  id bigint PRIMARY KEY,
  tenant_id int NOT NULL,
  doc_type text NOT NULL,
  body text NOT NULL,
  emb halfvec(1024) NOT NULL,          -- fp16: half the RAM, ~no recall loss
  emb_model text NOT NULL DEFAULT 'voyage-3.5@1024-norm'  -- versioned, enforced
);
CREATE INDEX ON chunks USING hnsw (emb halfvec_ip_ops)    -- ip: vectors are unit-norm
  WITH (m = 16, ef_construction = 100);

SET hnsw.ef_search = 80;                    -- tuned at the recall/latency knee
SET hnsw.iterative_scan = relaxed_order;    -- filtered queries keep scanning
SET hnsw.max_scan_tuples = 20000;           -- budget cap for pathological filters

SELECT id, body, -(emb <#> $1) AS score     -- <#> is NEGATIVE inner product
FROM chunks
WHERE tenant_id = $2 AND doc_type = 'policy'
ORDER BY emb <#> $1                          -- must match index opclass
LIMIT 10;
-- Verify with EXPLAIN ANALYZE that the HNSW index is used and the Filter
-- node isn't discarding >90% of fetched rows; if it is, partition by tenant
-- or add a partial index instead of raising max_scan_tuples forever.
```

**3. ANN-recall harness — index loss isolated from model loss:**
```python
import numpy as np

def exact_topk(q, X, ids, k=10):            # ground truth: brute force, same vectors
    return [ids[i] for i in np.argsort(-(X @ q))[:k]]   # unit-norm → dot = cosine

def ann_recall(queries, ann_search, X, ids, k=10):
    hits = tot = 0
    for q in queries:                        # 200+ REAL production queries
        truth = set(exact_topk(q, X, ids, k))
        got   = set(ann_search(q, k))        # same filters as production!
        hits += len(truth & got); tot += k
    return hits / tot

# Sweep the runtime dial; report the curve, run at the knee:
# ef_search:  40 → 0.91   80 → 0.962   160 → 0.985   320 → 0.991 (latency 2.1x)
```
If recall vs exact is ~0.99 but users still can't find things, the index is exonerated — the loss is in the model, chunking, or query handling. That separation is the entire point of this harness.

## Verification / self-check

Before presenting a vector-search design or diagnosis, confirm:

- **Arithmetic done:** raw vector bytes, index overhead, and RAM/disk placement computed for the actual N and D — not assumed.
- **Recall measured, twice:** ANN-vs-exact recall@k (index loss) and labeled-query recall@k (model loss), both **with production filters applied**, on real-usage-style queries.
- **Versioning enforced:** model ID + dims + normalization + prefix convention pinned to the index; mixed-vector states unreachable; a re-embed migration path exists (blue-green, budgeted).
- **Runtime dials tuned:** ef_search/nprobe/oversampling swept on your data; you know where the knee is and you're on it; quantized indexes rescore with full precision.
- **Score hygiene:** no absolute similarity thresholds; operator matches opclass (checked via EXPLAIN or engine equivalent); zero/NaN vectors rejected at ingest.
- **Fast-moving facts flagged:** model names, versions, leaderboard standings, and scale thresholds cited as "as of 2026" and re-verifiable — recommend re-checking rather than asserting stale specifics.
- **Stopping rule:** once filtered recall@k meets target at budget latency and memory fits with ≥30% headroom, stop tuning the index — remaining quality problems live upstream (model, chunking, query handling) and further index work is waste.
