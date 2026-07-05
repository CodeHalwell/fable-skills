---
name: llm-inference-optimization
description: Performance engineering for LLM serving — prefill/decode asymmetry, KV-cache memory arithmetic, continuous batching, quantization selection (FP8/INT4/NVFP4), speculative decoding, prefix caching, and GPU sizing from memory bandwidth. Load when sizing GPUs for a model, diagnosing slow TTFT/ITL or low throughput, choosing between vLLM/SGLang/TensorRT-LLM, picking a quantization format, or estimating tokens/sec and cost per token.
---

# LLM Inference Optimization

## The standard doctrine, compressed

A strong model already produces this cold; anchors only. Prefill compute-bound, decode bandwidth-bound — TTFT and ITL respond to different knobs; always ask "slow at which phase?" Decode roofline: tok/s ≤ bandwidth ÷ bytes/token (70 GB FP8 70B on H200 4.8 TB/s → ~68 ceiling, 75–85% realistic; 20 measured = config bug, 55 = done tuning kernels). KV/token = 2 × layers × kv_heads × head_dim × bytes (Llama-3.1-8B 128 KB, 70B 320 KB, MHA-70B would be 2.5 MB — why GQA exists). Throughput = KV-capacity problem: weights amortize across the batch, KV doesn't. FlashAttention/PagedAttention/continuous batching/chunked prefill are 2026 *defaults*, not upgrades. Quantization by regime: low-batch → weight-only INT4; high-batch → W8A8 FP8 (INT4 dequant makes it *slower* there); KV-pressure → FP8 KV first (cheapest 2× concurrency; calibrated scales, eval with needle tasks not perplexity). NVFP4 (16-elem blocks, E4M3 scales) beats MXFP4 (32-elem, E8M0) on Blackwell; production shape = NVFP4 weights + FP8/BF16 attention/KV. Speculative decoding: E[tokens/pass] = (1−α^(k+1))/(1−α); α=0.75,k=4 → ~3.05 → ~2.2× after overhead; a *latency* tool that reduces throughput at high batch; EAGLE-3-class heads are the standard (α≈0.75–0.8 greedy; temperature craters α; separate draft models are legacy — tokenizer alignment). TTFT-p99-high + ITL-spikes signature = preemption/KV pressure: check engine preemption warnings, fix `max_model_len`, cap concurrency, chunked-prefill budget. Engines: vLLM default, SGLang for shared-prefix/structured (RadixAttention), TensorRT-LLM for frozen-model all-NVIDIA last-15–30%. Parallelism: DP if it fits; smallest TP that fits within NVLink; PP across nodes; EP for MoE; the classic error is TP past fitting "for speed." `gpu_memory_utilization` 0.90–0.95, never 0.98. Goodput (SLO-meeting throughput) is the only optimizable number; benchmark with warm-up, mixed realistic lengths, client-side streaming TTFT, matched configs.

## Corrections and sharpenings

- **GPU spec table (verified as of 2026 — Opus-class models misremember B200):** H100 SXM 80 GB @ 3.35 TB/s; H200 141 GB @ 4.8 TB/s; **B200 180 GB @ 8 TB/s** (the reflex answer "192 GB" is the pre-production spec); MI300X 192 GB @ 5.3 TB/s; MI355X 288 GB @ 8 TB/s. Choose fewest-biggest GPUs that fit weights+KV after quantization, then bandwidth/$ for decode-heavy, FLOPs/$ for prefill-heavy. Buying by FLOPs for chat fleets is a seven-figure mistake: 2× FLOPs + 1.2× bandwidth serves chat 1.2× faster.
- **API prefix-cache pricing (as of 2026):** cached reads price at **~10% of input on both Anthropic and OpenAI flagships** — the common stale belief is that OpenAI's cached discount is only 50%; on current flagships it's ~90% off (e.g. $0.50 vs $5.00). Anthropic bills cache *writes* at 1.25× (5-min TTL) or 2× (1-hour). Break-even ≈ 2 reads per written prefix. Self-hosted: prefix caching is free-on by default; the design work is prompt layout — one timestamp at position zero kills every hit.
- **Capacity plan in one division:** 70B FP8 (~70 GB) on H200: ~60 GB for KV → at FP8 KV (160 KB/tok) ≈ 375k cached tokens ≈ **~45 concurrent 8k-context requests**. MLA (DeepSeek-class, ~70 KB/tok at 61 layers) is why those models batch so deep. MoE sizing: capacity on *total* params, batch-1 latency on *active*, high-batch throughput in between (many experts activate across a batch) — measured, not assumed.
- **The diagnosis prior, in order of hit rate:** (1) preemption/recompute warnings + KV pressure — the top cause of tail-latency incidents; (2) prefix-cache hit rate ≈ 0 (variable content at prompt head); (3) defaults mismatching the product (`max_model_len` at 128k reserves worst-case and thrashes); (4) benchmark methodology; (5) wrong regime for the optimization (INT4/spec-decode at high batch, deep TP); (6) only then kernels/engines/disaggregation. Most "slow inference" incidents are memory-pressure scheduling pathology, not slow math.
- **Prefill/decode disaggregation** earns its ops cost only with strict ITL p99 + mixed prompt lengths + multi-node scale + fast interconnect; below that, tune chunked prefill. KV-connector ecosystem as of 2026: NIXL, Mooncake, LMCache, **MoRI-IO on ROCm**. Chunked prefill: never disable it to "fix TTFT" — tune the chunk size against the ITL SLO instead.
- **Long-context TTFT is quadratic-ish:** attention FLOPs grow ~quadratically; a 100k prompt is not 10× a 10k one. Estimate prefill ≈ 2 × params × prompt_tokens (+ attention term) ÷ achievable FLOPs *before* promising interactive 128k.
- **CPU-bound small models:** an 8B on an H200 can outrun a single-process Python frontend; signature = throughput plateau + low GPU util + batch won't fill → fix API-server workers/tokenizer processes before touching the GPU. Structured/guided decoding adds per-token mask cost — a naive CPU backend doubles ITL; benchmark it as a first-class axis if JSON is core.
- **Quality-transfer trap:** "FP8 loses <1% MMLU" says nothing about a 20-step agent loop where per-step degradation compounds; eval quantized candidates on your end-to-end task. KV quantization damages long-range retrieval before perplexity moves.

## Worked arithmetic (the four calculations that settle most arguments)

```text
1. Roofline: 4.8 TB/s ÷ 70 GB ≈ 68 tok/s ceiling; 75–85% realistic → 51–58. Below 50% of roofline = config bug, not a hardware need.
2. Batching: step = weights once + Σ per-seq KV. FP8 KV @8k = 1.3 GB/seq; batch 32 → (70+42) GB/step → ~43 steps/s → ~1,370 tok/s aggregate (20× batch-1 for 1.6× ITL); memory wall at batch ~48 on 141 GB.
3. Speculation go/no-go: α=0.75,k=4 → 3.05/pass → ~2.2× real; α=0.5 → ~1.4× marginal; α=0.4 → off. Read measured acceptance from engine metrics before crediting anything.
4. Cost: H200 ≈ $3–4/hr → batch-32 ≈ 4.9M out-tok/hr ≈ $0.6–0.8/M output tokens self-hosted 70B; batch-1 ≈ $15–20/M — 25× spread. Utilization IS the price; redo with measured throughput before any build-vs-buy claim.
```

## Verification / self-check

- Roofline and KV budget computed; the recommendation changes bytes-moved, memory pressure, or scheduling — otherwise it changes nothing.
- Symptom localized to a phase (TTFT→prefill/queue/cache-miss; ITL level→bandwidth; ITL spikes→scheduling/preemption) and the fix targets that phase.
- Quantization advice names the regime, the hardware generation, and the use-case quality risk.
- Numbers are p50+p99, client-side streaming TTFT, at a stated request rate and length distribution; hit/acceptance rates read from engine metrics.
- Version-sensitive claims marked "as of 2026" — engine defaults, format support, and GPU specs turn over in months.
- Stopping rule: ~80% of roofline + SLOs met + no preemption warnings → done; below 50% of roofline → find the config bug before buying hardware or quantizing anything.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 11 baseline (cut/compressed — roofline + KV arithmetic exact, INT4/FP8 regimes, NVFP4 vs MXFP4 block structures, EAGLE-3 + acceptance math, preemption signature diagnosis, engine selection, parallelism chain, 0.98 memory-util, disaggregation connectors, benchmark pitfalls), 2 partial (sharpened), 0 fully wrong.
- Biggest baseline gaps: OpenAI cached-input discount miscalibrated (Opus says ~50%; flagships are ~90% off as of 2026); B200 memory misremembered as 192 GB (production is 180 GB). Both are load-bearing for capacity/cost math.
- Kept as likely-delta despite unprobed: the concurrency-budget division (~45×8k on H200), self-hosted $/Mtok spread, MoRI-IO/ROCm connector, diagnosis-prior ordering as an explicit hit-rate ranking.
