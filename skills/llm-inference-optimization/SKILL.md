---
name: llm-inference-optimization
description: Performance engineering for LLM serving — prefill/decode asymmetry, KV-cache memory arithmetic, continuous batching, quantization selection (FP8/INT4/NVFP4), speculative decoding, prefix caching, and GPU sizing from memory bandwidth. Load when sizing GPUs for a model, diagnosing slow TTFT/ITL or low throughput, choosing between vLLM/SGLang/TensorRT-LLM, picking a quantization format, or estimating tokens/sec and cost per token.
---

# LLM Inference Optimization

## Core mental model

1. **Prefill and decode are two different computational regimes; every optimization helps one and often hurts the other.** Prefill processes all prompt tokens in one pass — big matmuls, high arithmetic intensity, **compute-bound** (saturates tensor cores). Decode emits one token per sequence per step — the entire weight matrix and the sequence's whole KV cache stream from HBM to produce a single token, so arithmetic intensity is ~1 FLOP/byte and it's **memory-bandwidth-bound**. This asymmetry is the master key: TTFT is a prefill/compute problem, ITL is a decode/bandwidth problem, and they respond to different knobs. When someone says "inference is slow," your first question is always: slow at which phase?
2. **Decode speed has a hard roofline: tokens/sec ≤ memory bandwidth ÷ bytes moved per token.** At batch 1, bytes per token ≈ the model's weight footprint (plus that sequence's KV). No kernel, no framework, no clever scheduling beats this number — only moving fewer bytes does (quantization, MoE sparsity, speculative decoding's parallel verification). Do this division before any benchmark; if measured throughput is within ~75–85% of the roofline, kernel-level tuning is finished and further gains must come from reducing bytes.
3. **Throughput is a KV-cache memory problem.** Batching decode requests amortizes the weight read across the batch nearly for free (weights stream once per step regardless of batch size), so throughput scales with concurrency — until KV cache exhausts GPU memory. Max concurrency ≈ (VRAM − weights − activations) ÷ (KV bytes/token × avg context length). This is why GQA/MQA/MLA, KV quantization, and paged memory matter more for serving economics than any FLOPs number.
4. **The 2026 baseline already includes the famous optimizations.** FlashAttention-class kernels, PagedAttention/paged KV, continuous (in-flight) batching, and chunked prefill are the *defaults* in vLLM, SGLang, and TensorRT-LLM — not upgrades you add. If a deployment lacks them, it's misconfigured, not unoptimized. The remaining levers you actually decide on: quantization format, speculative decoding, prefix caching policy, parallelism layout, prefill/decode disaggregation, and scheduler limits.
5. **There is no single "latency": TTFT, ITL, and throughput trade against each other.** Raising batch size raises throughput and ITL together. Prioritizing an incoming prefill improves that request's TTFT and stalls everyone else's ITL. Never optimize or report one number; state the SLO triple (e.g., "TTFT p99 < 1s, ITL p99 < 60ms, at N req/s") and optimize *goodput* — throughput of requests that meet the SLO — not raw tokens/sec.
6. **Napkin arithmetic before benchmarks.** Weight bytes, KV bytes/token, HBM bandwidth, and FLOPs fit on an index card and predict real systems within ~25%. An expert computes the expected number first, then benchmarks to find the gap; a generalist benchmarks first and has no idea whether 40 tok/s is great or a bug.

## KV-cache arithmetic (the calculation everything else depends on)

```
KV bytes/token = 2 (K and V) × n_layers × n_kv_heads × head_dim × bytes_per_elem
```

Worked for real models at BF16 (2 bytes), configs verified against the released checkpoints:

| Model | layers × kv_heads × head_dim | KV/token | 32k-token request |
|---|---|---|---|
| Llama-3.1-8B (GQA, 8 kv heads) | 32 × 8 × 128 | 128 KB | 4.0 GB |
| Llama-3.1-70B (GQA, 8 kv heads; 64 Q heads) | 80 × 8 × 128 | 320 KB | 10.2 GB |
| Same 70B if it were MHA (64 kv heads) | 80 × 64 × 128 | 2.5 MB | 82 GB — why nobody ships MHA anymore |
| DeepSeek-V3-class (MLA: 576-dim latent/layer, 61 layers) | — | ~70 KB | 2.2 GB |

Reasoning chain this table drives:
- **GQA divides KV by (n_q_heads/n_kv_heads)** — 8× for Llama-70B. MQA (1 kv head) is the extreme. MLA compresses K and V jointly into a low-rank latent (~4–5× smaller than even GQA), which is why DeepSeek-style models batch so deep and serve long context cheaply.
- **FP8 KV cache halves all of these** (supported in vLLM/SGLang/TensorRT-LLM as of 2026; use a calibrated/scaled variant, not naive casting).
- Concurrency budget, e.g. Llama-3.1-70B in FP8 weights (~70 GB) on one H200 (141 GB): ~60 GB left for KV after activations/overhead → at FP8 KV (160 KB/token) ≈ 375k cached tokens ≈ **~45 concurrent 8k-context requests**. That single division is your capacity plan.

## Decision frameworks

**Which engine (as of mid-2026):**
- Default to **vLLM** — broadest model/hardware coverage, fast starts, continuous batching + paged KV + chunked prefill + prefix caching on by default; FlashAttention backend on Hopper, FlashInfer on Blackwell.
- Choose **SGLang** when the workload has heavy shared prefixes (multi-turn chat over a long system prompt, agent loops, RAG over repeated context) — RadixAttention's tree-structured prefix reuse is its edge — or heavy structured/constrained generation.
- Choose **TensorRT-LLM** when the fleet is all-NVIDIA, the model is frozen, and you'll pay an engine-compile step and operational complexity for the last 15–30% throughput; it also has the most mature NVFP4 path on Blackwell.
- Reasoning order: start vLLM → profile → move only when a measured bottleneck names one of the alternatives' specific strengths. Migrating engines on vibes is a week lost to re-benchmarking.

**Quantization decision chain — ask in this order:**
1. *What's actually the bottleneck?* Batch-1/low-batch decode → weights dominate bytes → **weight-only 4-bit** (AWQ/GPTQ-class) gives near-linear ITL speedup. High-batch serving → compute and KV dominate → weight-only INT4 barely helps and dequant overhead can make it *slower* than FP8; use **W8A8 FP8** (weights *and* activations) to engage FP8 tensor cores — measured ~1.5–1.6× serving throughput vs BF16 at moderate QPS. KV memory capping concurrency → **FP8 KV cache** first, it's the cheapest 2× concurrency you'll ever buy.
2. *What hardware?* FP8 needs Hopper/Ada or newer. Blackwell (B200/B300-class) adds native FP4 tensor cores: **NVFP4** (16-value blocks, E4M3 per-block scales + FP32 tensor scale) is the quality-leading 4-bit there; MXFP4 (32-value blocks, power-of-two scales) quantizes noticeably worse. Common production shape on Blackwell as of 2026: NVFP4 weights with FP8/BF16 attention and KV — pure-FP4 everywhere loses measurable quality.
3. *What's the use case's quality tolerance?* Chat/summarization tolerates 4-bit weights easily. Code, math, multi-step agentic tool use, and long-context retrieval degrade first — FP8 is typically indistinguishable from BF16 on these (fractions of a point on MMLU-Pro-class evals), INT4 loses several points on code (HumanEval-class). KV-cache quantization specifically hurts long-context *retrieval* accuracy before anything else — eval "needle" tasks, not just perplexity, before shipping quantized KV.
4. Stopping rule: quantize until the bottleneck moves (bandwidth → compute, or memory → bandwidth), then stop. Stacking more quantization past that point spends quality for nothing.

**Speculative decoding — when it pays:**
- Mechanism: a cheap drafter proposes k tokens; the target model verifies all k in *one* forward pass (prefill-like, parallel). You convert decode's idle compute into extra tokens per weight-read. With per-token acceptance rate α and k draft tokens, expected tokens per target pass ≈ (1 − α^(k+1)) / (1 − α). At α = 0.75, k = 4: ≈ 3.05 tokens/pass → roughly 2–2.5× after draft overhead.
- Pays when: memory-bound with spare compute — i.e., **low batch, latency-sensitive** traffic. Hurts when: batch is already large enough to be compute-bound (verification FLOPs now displace other requests' work — spec decode at high batch *reduces* throughput); or output distribution is high-entropy (high temperature, creative writing → α collapses; greedy/code/structured output → α is highest).
- What to use (as of 2026): **EAGLE-3-class draft heads** are the production standard in vLLM, SGLang, and TensorRT-LLM — target-attached heads reading the target's hidden states, acceptance ~75–80% on typical workloads. MTP-style self-drafting applies when the model was trained for it (DeepSeek lineage). Separate small draft *models* are legacy: tokenizer must match exactly and acceptance is usually worse.
- N-gram/prompt-lookup speculation is free and shockingly effective when output copies input (editing, summarization-with-quotes, code refactoring) — try it before training anything.

**Prefix caching — the economics:**
- Self-hosted: enable it, always (vLLM automatic prefix caching; SGLang RadixAttention). Cached prefixes skip prefill compute — TTFT drops roughly proportionally to the cached fraction, and it costs only KV memory that paged eviction reclaims under pressure. There is no downside at typical hit rates; the design work is *prompt layout*, not the cache itself: shared, stable content (system prompt, tool schemas, few-shot examples) first, per-request content last. One timestamp or user ID at position zero kills every hit.
- API providers (as of 2026): cached input reads price at ~10% of normal input on both Anthropic and OpenAI flagships; Anthropic bills cache *writes* at 1.25× input. Break-even is ~2 reads per written prefix — cache anything reused, don't cache one-shot prompts.

**Prefill/decode disaggregation:**
- Colocated prefill and decode contend: an arriving long prompt's prefill inflates other requests' ITL (chunked prefill bounds but doesn't eliminate this). Disaggregation runs prefill and decode on separate instances/GPUs, shipping KV across (vLLM KV-connectors: NIXL, Mooncake, MoRI-IO on ROCm — as of 2026), letting you scale and parallelize each phase to its own SLO: TTFT capacity = prefill pool, ITL stability = decode pool.
- Reasoning: worth it when (a) strict ITL p99 SLOs with mixed prompt lengths, (b) enough scale that separate pools stay utilized, and (c) fast interconnect for KV transfer. Below multi-node scale, tune chunked prefill and scheduler limits first — disaggregation is an ops-complexity purchase.

**Latency metrics — definitions and which knob moves which:**
- **TTFT** (time to first token) = queueing + prefill. Moved by: prefix caching (skips prefill of cached tokens), more prefill compute (FLOPs, prefill pool size), admission control (shorter queues), chunked-prefill chunk size. NOT moved by: speculative decoding, faster decode kernels.
- **ITL / TPOT** (inter-token latency / time per output token) = decode step time. Moved by: fewer bytes per step (quantized weights/KV), smaller batch, speculative decoding, avoiding prefill interference. NOT moved by: prefix caching (already past prefill).
- **Throughput** (aggregate tok/s) — moved by batch depth, which is moved by KV capacity. Rises together with ITL; the batch-size knob slides along that tradeoff curve, it cannot improve both.
- **Goodput** — requests/s meeting the full SLO triple. The only number worth optimizing; a config that raises throughput 30% while pushing ITL p99 past SLO has *negative* value.
- Reasoning habit: for any proposed change, say out loud which of these four it moves, in which direction, at whose expense. If you can't, you don't understand the change yet.

**Parallelism layout — reasoning chain:**
1. Does the model (+ KV target) fit on one GPU after quantization? If yes, run **data-parallel replicas** — zero interconnect tax, linear throughput scaling, simplest ops.
2. If not: **tensor parallel** at the smallest degree that fits, within one node (NVLink domain). TP divides both weights and KV per GPU and multiplies effective bandwidth, but adds two allreduces per layer — latency cost grows with TP degree and becomes prohibitive across nodes.
3. **Pipeline parallel** only when a model exceeds a node (very large dense models): no per-layer collectives, but adds pipeline bubbles and per-stage latency; prefer it across nodes, TP within nodes.
4. MoE models add **expert parallelism** (experts sharded across GPUs, all-to-all routing per layer); watch expert load imbalance — a hot expert serializes the batch. As of 2026 the large-MoE pattern is EP+DP for decode with wide EP (DeepEP-style kernels), TP mostly for attention.
5. Prior: the most common layout error is *too much* TP "for speed" — TP is a memory-fitting tool that happens to add bandwidth, and past fitting, collectives eat the gains.

**GPU selection arithmetic:** memory bandwidth is the binding constraint for decode; capacity bounds concurrency. Verified specs (as of 2026): H100 SXM 80 GB @ 3.35 TB/s; H200 141 GB @ 4.8 TB/s; B200 180 GB @ 8 TB/s; AMD MI300X 192 GB @ 5.3 TB/s; MI355X 288 GB @ 8 TB/s. Choose by: (1) do weights + working KV fit (after quantization) in as few GPUs as possible — fewer, bigger GPUs beat more, smaller ones because tensor parallelism taxes every layer with interconnect collectives; (2) then rank by bandwidth/$ for decode-heavy workloads, FLOPs/$ for prefill-heavy (long-prompt RAG, batch embedding/summarization).

## How an expert thinks through this

Scenario: "Our Llama-3.1-70B chat service on 4×H100 (TP=4) feels sluggish; users see slow responses. Make it faster."

*First, refuse the vague framing — decompose the symptom.* Pull metrics: TTFT p50/p99 and ITL p50/p99. Suppose: TTFT p99 = 6s, ITL p50 = 45ms, ITL p99 = 400ms. So decode is mostly fine (45ms ≈ 22 tok/s streaming, readable), but two things are wrong: first-token wait, and ITL *spikes*. Those two symptoms co-occurring is a signature: long prefills are stalling the decode batch — a scheduling problem, not a speed problem.

*Check the roofline anyway to calibrate.* 70B BF16 = 140 GB weights across 4×3.35 TB/s = 13.4 TB/s aggregate → batch-1 ceiling ≈ 13,400/140 ≈ 95 tok/s, minus TP allreduce overhead. ITL p50 of 45ms (22 tok/s) at real batch sizes is plausible — no kernel bug. Don't touch kernels.

*Consider and reject, in order:*
- "Add more GPUs / go TP=8." Rejected: TTFT p99 isn't a capacity problem if p50 is fine — check queueing first. Also TP=8 across nodes would add interconnect latency to every layer.
- "Switch engines." Rejected: no evidence any engine-specific strength is the bottleneck. Re-benchmarking tax for nothing.
- "Speculative decoding." Tempting — but it fixes ITL *level*, and our p50 ITL is acceptable; it does nothing for TTFT or tail spikes. Park it.
- "Quantize to FP8." Genuinely relevant — halves weight bytes (ITL headroom) and, more importantly here, halves weight memory: 140 GB → 70 GB frees ~70 GB for KV, roughly doubling max concurrency, which shrinks the queue that's inflating TTFT p99. Quality risk low (FP8 ≈ BF16 on chat evals). Accept, but it's step two.
- Step one: *look at the scheduler logs.* Found: preemption warnings — KV cache pressure is forcing requests to be evicted and *recomputed*, which explains both ITL spikes (recompute looks like a giant prefill) and TTFT tail (queue). Root cause candidates: `max_model_len` left at the model's 128k default (so the scheduler reserves for worst-case), too-high concurrency limit, no prefix caching across turns of the same conversation.

*Plan, cheapest first:* (1) set `max_model_len` to the real product limit (say 16k) and cap concurrent sequences to what KV math supports; (2) confirm prefix caching is on — multi-turn chat re-prefills the whole history every turn without it, and that's likely half the prefill load; (3) FP8 weights + FP8 KV to double both bandwidth headroom and KV capacity; (4) re-measure the SLO triple; (5) only if TTFT p99 still fails at target load, split traffic or disaggregate prefill. Expected: steps 1–3 fix it; most "slow inference" incidents are memory-pressure-induced scheduling pathology, not slow math. That's the prior: **check KV pressure and preemption before anything with "kernel," "compile," or "speculative" in the name.**

## Failure modes & pitfalls

- **Quoting batch-1 tokens/sec as "throughput."** Batch-1 measures the bandwidth roofline; production throughput is 10–100× higher via batching. Conversely, quoting max-batch aggregate throughput while hiding 300ms ITL. Always report the triple at a stated request rate and context-length distribution.
- **Benchmarking with uniform, short, identical prompts.** Identical prompts all hit the prefix cache (numbers 5× too good); uniform lengths hide the chunked-prefill/ITL interference that mixed real traffic causes; short contexts hide KV pressure. Replay a production trace or synthesize matched length distributions.
- **Weight-only INT4 deployed to a high-batch service to "increase throughput."** At high batch the system is compute-bound; INT4 weights must be dequantized before matmul, adding compute. Result: same or *lower* throughput than FP8 W8A8, plus quality loss. INT4's home is low-batch/VRAM-constrained decode; FP8's home is high-batch serving. Match format to regime, not "smaller = faster."
- **Naive-cast FP8/INT8 KV cache shipped after checking only perplexity.** KV quantization error accumulates over context; it degrades long-range retrieval ("what did the user say 40k tokens ago") while perplexity barely moves. Use calibrated scales and evaluate with needle/long-context tasks before shipping.
- **Speculative decoding enabled fleet-wide, throughput drops.** At high batch, verification FLOPs displace other requests' decode work; spec decode is a *latency* tool for the low-batch regime. Also check sampling: temperature ≥ 1 craters acceptance rate; if acceptance < ~60%, the draft overhead exceeds the win. Measure accepted-tokens-per-step in the engine's metrics, don't assume paper numbers.
- **Draft model with a mismatched tokenizer (or vocab).** Classic separate-draft-model failure: token IDs must align exactly for verification. EAGLE-class heads sidestep this — another reason they won.
- **Prefix cache hit rate silently ~0%.** Someone put `Current time: 2026-07-05T14:03:22` (or a request UUID, or the user's name) at the top of the system prompt. Every request's prefix differs from token one. Order prompts: static system content → tools/few-shots → session history → per-request input. Check the engine's reported hit rate; don't infer it.
- **`max_model_len` / context limit left at the model's maximum.** Reserving for 128k contexts your product never uses slashes admitted concurrency and triggers preemption thrash — evicted requests get *recomputed*, which shows up as mysterious ITL spikes and TTFT tail. Watch for the engine's preemption/recompute warnings; they're the smoking gun of most production latency incidents.
- **Tensor parallelism treated as a free bandwidth multiplier.** TP=8 gives ~8× aggregate bandwidth but adds two allreduces per layer; past the point where the model fits, more TP often *raises* ITL. Never TP across node boundaries for latency-sensitive decode. Prefer the smallest TP that fits weights + KV target; scale throughput with data-parallel replicas (or disaggregation), not deeper TP.
- **MoE sizing done on active parameters.** A DeepSeek-V3-class MoE with ~37B active / ~671B total params: *all* weights must sit in HBM (capacity is total), while batch-1 decode streams roughly the active subset (bandwidth is active-ish) — but at high batch, many experts activate across the batch and you stream most of the model again. Capacity-plan on total params; batch-1 latency on active; high-batch throughput somewhere in between, measured.
- **Ignoring where the SLO metric is measured.** TTFT measured server-side without streaming enabled includes zero queueing/network and can be 10× lower than what users see; client-measured with streaming is the only honest TTFT. Similarly "average latency" hides everything — decode incidents live in ITL p99.
- **Comparing engines at different defaults and concluding one is faster.** Engine A with FP8 + prefix caching + chunked prefill vs engine B at BF16 defaults is a config comparison, not an engine comparison. Pin model precision, KV precision, max batch, scheduler limits, and cache settings before believing any cross-engine benchmark (including published ones).
- **Buying GPUs by FLOPs for a decode-dominated workload.** A GPU with 2× FLOPs and 1.2× bandwidth serves chat ~1.2× faster. Read the workload first: chat/agents are decode-heavy (bandwidth); document-ingestion/RAG-indexing/batch-scoring are prefill-heavy (FLOPs). Fleet mistakes here are seven-figure mistakes.
- **Fixing TTFT by disabling chunked prefill.** It "works" for the request you're staring at and destroys ITL for the batch it stalls. Chunked prefill exists to bound decode starvation; tune the chunk size against your ITL SLO instead of turning it off.
- **Assuming quantization quality claims transfer across use cases.** "FP8 loses <1% on MMLU" says nothing about your agentic tool-calling loop, where small per-step degradation compounds across 20 steps. Eval quantized candidates on *your* end-to-end task with *your* prompts; benchmark deltas are a prior, not a verdict.
- **Small model on big GPU bottlenecked on CPU, blamed on the GPU.** An 8B model on an H200 can decode faster than a single-process Python frontend can tokenize/detokenize/schedule; GPU utilization sits at 40% while everyone tunes kernels. Signature: throughput plateaus while GPU util is low and batch won't fill. Fix the serving process (more API-server workers, separate tokenizer processes) before touching anything on the GPU.
- **Benchmarking before warm-up.** First requests pay CUDA-graph capture, kernel autotuning/JIT, and (TensorRT-LLM) engine load; including them makes TTFT look catastrophic and steady-state numbers noisy. Warm the server with representative shapes, then measure. Conversely, don't let a 28-minute TensorRT-LLM compile surprise you at deploy time — it's a known cost you schedule, not a regression.
- **Guided/structured decoding overhead unaccounted.** JSON-schema/grammar-constrained generation adds per-token mask computation; a naive backend serializes it on CPU and doubles ITL. If structured output is a core workload, this is a first-class benchmark axis (and a reason SGLang gets picked), not a checkbox.
- **`gpu_memory_utilization` pushed to 0.98 to "use all the memory."** Long-context spikes, CUDA graph pools, and fragmentation need slack; the failure is an OOM crash hours into production, taking every in-flight request with it. 0.90–0.95 is the sane band; buy concurrency with KV quantization and `max_model_len`, not by shaving the safety margin.
- **Long-context TTFT surprise: prefill is quadratic in the attention term.** A 100k-token prompt isn't 10× a 10k prompt — attention FLOPs grow ~quadratically with sequence length even though the MLP term is linear, and TTFT for six-figure contexts runs to many seconds regardless of engine. Budget it: estimate prefill FLOPs ≈ 2 × params × prompt_tokens (plus attention term), divide by achievable FLOPs, and check against the TTFT SLO *before* promising 128k support in an interactive product.

## Worked micro-examples

**1. Decode roofline — Llama-3.1-70B, FP8 weights (~70 GB), single H200 (4.8 TB/s):**
```
batch-1 ceiling = 4,800 GB/s ÷ 70 GB/token-step ≈ 68 tok/s
realistic       ≈ 75–85% of ceiling ≈ 51–58 tok/s   (kernel + overhead losses)
```
If you measure 20 tok/s here, something is broken (wrong backend, no FP8 kernels, CPU bottleneck); if you measure 55, stop tuning kernels — only fewer bytes (INT4/NVFP4 weights, speculation) go faster. Same model on B200 (8 TB/s): ceiling ≈ 114 tok/s — hardware bandwidth is the cheapest "optimization" there is.

**2. Batching arithmetic on the same box — why throughput scales, and what stops it:**
```
Per decode step: read weights once (70 GB, shared) + each sequence's KV.
FP8 KV, 8k context: 8,192 tok × 160 KB ≈ 1.3 GB per sequence.
Batch 32: (70 + 32×1.3) GB = 112 GB/step → 4,800/112 ≈ 43 steps/s
  → aggregate ≈ 1,370 tok/s (vs 68 at batch 1 — 20× throughput for 1.6× ITL)
Memory check: 70 (weights) + 42 (KV) + overhead ≈ 120 GB < 141 GB ✓; batch ~48 is the wall.
```
The two lessons in one table: weights amortize (throughput scales), KV doesn't (it caps batch and eventually dominates the step time — which is exactly why GQA→MLA and FP8 KV are throughput features, not memory conveniences).

**3. Speculative decoding acceptance math — go/no-go in 30 seconds:**
```
E[tokens per target pass] = (1 − α^(k+1)) / (1 − α)
α = 0.75, k = 4  → (1 − 0.237)/0.25 ≈ 3.05 tokens/pass
Draft + verify overhead ≈ 25–30%  → real speedup ≈ 3.05 × 0.72 ≈ 2.2×
α = 0.5 (chatty high-temp traffic), k = 4 → 1.94 tokens/pass → ≈ 1.4× — marginal
α = 0.4 → 1.7 raw → ≈ 1.2× — likely a loss after tail effects; turn it off
```
EAGLE-3-class heads hit α ≈ 0.75–0.8 on greedy/low-temp workloads (as of 2026). Read your engine's measured acceptance metric and rerun this formula before crediting any speedup claim.

**4. Cost per million output tokens — from the batching example, not from vibes:**
```
H200 on-demand ≈ $3–4/hr (2026 cloud ballpark; substitute your real rate).
Batch-32 aggregate from example 2: ≈ 1,370 tok/s ≈ 4.9M output tok/hr
→ ≈ $0.6–0.8 per 1M output tokens for self-hosted 70B-class FP8.
Batch-1 (dedicated latency box): 55 tok/s ≈ 0.2M tok/hr → ≈ $15–20 per 1M — 25× worse.
```
This spread is the whole economics of inference: utilization *is* the price. It's why
batch APIs are cheap, why providers fight for prefix-cache hits, and why "we'll just
self-host" is only true for teams that can keep batches full. Redo this arithmetic with
your measured aggregate throughput before making any build-vs-buy claim.

## Verification / self-check

Before presenting an inference-performance answer, confirm:
- You computed the roofline (bandwidth ÷ bytes/token) and the KV budget ((VRAM − weights) ÷ KV-per-token) and your recommendation is consistent with both. If your advice doesn't change either bytes-moved or memory-pressure or scheduling, it won't change performance.
- You identified *which phase* the symptom lives in (TTFT→prefill/queueing/cache-miss; ITL level→bandwidth; ITL spikes→scheduling/preemption/prefill interference) and the fix targets that phase.
- Any quantization recommendation names the regime (low-batch vs high-batch), the hardware generation it needs, and the use-case quality risk — and you did not claim INT4 speeds up compute-bound serving.
- Metrics discipline: numbers are p99 + p50 with streaming, client-side for TTFT, at a stated request rate and context distribution; hit rates and acceptance rates read from engine metrics, not assumed.
- Version-sensitive claims (engine defaults, format support, GPU specs) are marked "as of 2026" and came from checking, not memory — this landscape turns over in months.
- Stopping rule: within ~80% of roofline with SLOs met and no preemption warnings → done; further optimization is spending engineer-weeks to buy single-digit percents. Below 50% of roofline → there is a config bug; find it before buying hardware, changing engines, or quantizing anything.
