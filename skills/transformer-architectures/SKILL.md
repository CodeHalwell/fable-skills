---
name: transformer-architectures
description: Load when reasoning about transformer internals for engineering decisions — attention cost, KV-cache and inference memory sizing, MQA/GQA choices, positional encodings and context extension, parameter/FLOPs estimation from a config, MoE routing, layernorm placement, or choosing encoder-only vs decoder-only vs encoder-decoder.
---

# Transformer Architectures for Engineering Decisions

Assumed baseline (verified strong; don't re-derive in prose): KV bytes = 2·L·n_kv·head_dim·len·batch·bytes; params ≈ L·12d² + V·d with SwiGLU = 3 MLP matrices; decode is bandwidth-bound (tokens/s ≈ BW / bytes-read, FLOPs irrelevant at batch 1); FlashAttention is exact and O(n²) FLOPs, only IO/memory change; RoPE extension = PI/NTK/YaRN/θ-raise + finetune, validated with needle probes *and* a short-context regression check; GQA-8 default, MQA's single KV head replicates across TP ranks; spec-decode E[tokens/pass] = (1−α^{k+1})/(1−α); pre-LN stable / post-LN warmup-fragile, pre-LN residual-norm growth mitigated by QK-norm/z-loss; sliding window reach ≈ w×L through depth; StreamingLLM sink tokens; 2-way TP ≈ 1.5–1.8× decode at batch 1 (two all-reduces/layer), memory tool first.

## The discipline that is the actual delta

Most wrong answers in this domain are not knowledge gaps but skipped arithmetic. Rules:
- **Never answer from the model's marketing name.** Read config.json first; these fields flip conclusions: `num_key_value_heads` (all KV math), `intermediate_size` + gated act (3 vs 2 MLP matrices), `tie_word_embeddings`, `rope_theta`/`rope_scaling` (native vs extended context — large θ ≥500k or a scaling dict means the advertised max is an extension; verify quality at the far end), `sliding_window` (KV growth caps at w; long-range retrieval claims need per-layer checking), `num_local_experts`/`num_experts_per_tok` (memory tracks total params, compute tracks active), `torch_dtype`.
- **Sanity-gate your estimate against the advertised size (±15%)**; if off, you misread tied embeddings, GQA, or d_ff.
- **State the regime before recommending**: prefill (compute-bound) vs decode (bandwidth-bound); n vs 6d — attention FLOPs fraction ≈ n/(6d+n), so a 7B (d=4096) at 2k context is ~8% attention. Recommending FlashAttention-style fixes for a short-context latency problem is the classic regime error.
- If you can't derive a claimed speedup ratio in one line, don't claim it.

## Batched decode throughput (correcting a common muddle)

Weights are read **once per decode step for the whole batch**, not per sequence. So at batch B: per-stream tok/s ≈ BW / (weight_bytes + per-stream KV share), and **aggregate tok/s ≈ B × per-stream** until you hit the compute roof or KV bandwidth dominates. Example: 70B int8 on 2×80GB TP2 (35 GB weights/GPU, ~2 TB/s): ~17.5 ms weight read + ~5 ms KV (batch 8 × 8k, GQA-8: 320 KB/token → 10 GB/GPU) + all-reduce ≈ ~22–25 ms/step → ~40–45 tok/s **per sequence** and ~350 tok/s aggregate at batch 8. Batching is nearly free throughput in the bandwidth-bound regime — that, not per-request speed, is why continuous batching wins.

## Decision one-liners (completeness checklist; reasoning assumed known)

- Slow batch-1 decode → quantize weights (near-linear win), distill, speculative decoding (compute measured α first; loses at large batch where the target is already compute-bound, and on high-entropy generation).
- Decode degrades as batch grows → KV-bound: GQA if you can finetune, KV fp8/int8 (usually cheaper quality-wise than halving KV heads again), paged attention (fragmentation/max-batch, not bandwidth), sliding window if task tolerates.
- High TTFT on long prompts → prefill: prefix caching (shared system prompts — the big practical win), chunked prefill, FlashAttention.
- Throughput fine, tail latency bad → continuous (token-level) batching, not request-level.
- MoE: quality-per-FLOP, costs total-param memory + all-to-all latency at small batch; expert-collapse needs load-balancing losses; **capacity-factor overflow silently drops/bypasses tokens under imbalance — check router load stats, not loss**.
- Encoder-decoder: cross-attention KV is computed **once** from the encoder output and reused every decode step — different serving math from self-attn cache. Decoder-only won on training efficiency (every position is a target) and scaling simplicity, not representational superiority.
- Compressed-KV / multi-head-latent schemes: reduce every attention variant to (cache bytes/token, FLOPs/token, exact-retrieval range) before comparing — cache-bytes/token is the comparable axis.

## Numbers to have loaded (no derivation needed at use time)

- Llama-2-7B-class MHA: 512 KB/token → 2 GB per 4k stream; batch 16 = 32 GB (cache > weights). GQA-8 → 128 KB/token.
- Training ≈ 6ND FLOPs; decode ≈ 2N/token but bandwidth-ruled; training memory with Adam mixed precision ≈ 16 bytes/param (7B ≈ 112 GB before activations) — quoting 2 bytes/param for training is the classic 8× error.
- 7B fp16 on 1 TB/s: ~70 tok/s batch-1 ceiling.
- ALiBi extrapolates in perplexity but recency bias hurts exact long-range retrieval; RoPE+rescale is the retrieval-safe path. "Supports 128k" ≠ uses 128k — effective context degrades before the hard limit.

## Verification / self-check

1. Recompute params (L·12d² + Vd) and KV (2·L·kv·head_dim·len·bytes); check ±15% vs advertised.
2. Name the regime (prefill/decode, n vs 6d, memory- vs compute-bound) and confirm the recommendation matches.
3. Confirm config fields (tied embeddings, kv heads, d_ff/gating, rope_theta, sliding_window) before concluding.
4. Context-extension advice must include the validation probe, and batched-throughput claims must state per-stream vs aggregate explicitly.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (compressed to anchors/one-liners), 1 partial, 0 delta.
- Baseline was expert-grade across all arithmetic (KV, params, spec-decode formula, attention sinks, TP costs); the audit found the *skill's own* worked example understated aggregate batched-decode throughput (weights read once per batch step) — corrected here.
- This skill's residual value is process discipline (config-first, regime-first, derive-before-claiming) and the loaded reference numbers, not facts Opus lacks.
