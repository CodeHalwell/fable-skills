---
name: transformer-architectures
description: Load when reasoning about transformer internals for engineering decisions — attention cost, KV-cache and inference memory sizing, MQA/GQA choices, positional encodings and context extension, parameter/FLOPs estimation from a config, MoE routing, layernorm placement, or choosing encoder-only vs decoder-only vs encoder-decoder.
---

# Transformer Architectures for Engineering Decisions

## Core mental model

- **Attention is a soft key-value lookup.** Each query vector retrieves a convex combination of value vectors, weighted by softmax(q·k/√d). Everything downstream follows: it is permutation-invariant (hence positional encodings), it is content-addressed (hence in-context learning), and every query attends to every key (hence O(n²) time and O(n) KV memory per new token). When someone asks "why is long context expensive," the answer is this lookup, not the MLPs.
- **The residual stream is the backbone; blocks are read-write patches.** Each attention/MLP block reads from the stream, computes a delta, and adds it back. This is why you can skip, prune, or ablate individual layers with graceful degradation, why depth scales trainably, and why "what does layer 17 do" is a question about what it *writes into the stream*.
- **Inference has two regimes with different bottlenecks.** Prefill is compute-bound (big matmuls over the whole prompt); decode is memory-bandwidth-bound (each token re-reads all weights + the KV cache). Almost every inference optimization (quantization, GQA, speculative decoding, batching) is best understood as attacking decode bandwidth.
- **Parameter count and FLOPs are back-of-envelope computable from the config.** Never guess. Compute them (formulas below); errors of 2x in memory or FLOPs estimates are the most common cause of bad hardware decisions.
- **Most "architecture" questions are actually arithmetic questions.** Will it fit? How fast? How much cache? Do the multiplication before offering opinions.

## Attention and its O(n²) consequence

- Self-attention cost per layer: time ∝ n²·d for the QK^T and AV matmuls, plus n·d² for the projections. For short sequences (n ≪ d) projections dominate; the quadratic term only dominates when n is several times d. A 7B model at n=2k is *not* attention-dominated; at n=128k it is. Check which regime you're in before recommending FlashAttention-style fixes vs. just bigger batches.
- FlashAttention changes memory (no n×n matrix materialized, O(n) activation memory) and wall-clock (IO-aware tiling), **not** asymptotic FLOPs. Do not describe it as "linear attention."
- Sparse/linear attention variants trade recall of arbitrary long-range pairs for cost. Prefer exact attention until the KV cache or prefill latency is the demonstrated bottleneck.

## KV-cache arithmetic (memorize this)

Per token, per layer: 2 (K and V) × n_kv_heads × head_dim × bytes.
Total: **KV bytes = 2 × n_layers × n_kv_heads × head_dim × seq_len × batch × bytes_per_elt**

Worked example — Llama-2-7B-style config (32 layers, 32 heads, head_dim 128, MHA, fp16):
- Per token: 2 × 32 × 32 × 128 × 2 B = 512 KB/token.
- 4k context, batch 1: 2 GB. Batch 16: 32 GB — the cache, not the 13 GB of weights, kills your batch size.
- Same model with GQA n_kv_heads=8: 128 KB/token → 8 GB at batch 16. This is *the* reason GQA exists.

Decision rules:
- **MHA → GQA (4–8 KV heads):** near-lossless in quality at scale, 4–8× cache reduction; the default for any model meant to serve long contexts or big batches.
- **MQA (1 KV head):** maximal cache savings but measurable quality loss and awkward tensor-parallel sharding (the single KV head must be replicated across TP ranks). Choose only when cache is the overwhelming constraint.
- Quantizing KV cache to fp8/int8 is usually cheaper quality-wise than halving KV heads again.

## Positional encodings and length extrapolation

| Family | Mechanism | Extrapolation behavior | Use when |
|---|---|---|---|
| Learned absolute | trained embedding per position | none — positions past training length are literally untrained parameters | legacy/BERT-era; avoid for new decoders |
| Sinusoidal | fixed frequencies added to input | defined at any length but attention patterns still break OOD | rarely chosen now |
| RoPE | rotates q,k by position-dependent angle; attention depends on relative offset | breaks past trained length, but *fixable post-hoc* by rescaling frequencies (position interpolation / NTK-aware / YaRN-style) with little or no finetuning | default for decoder LMs |
| ALiBi | linear distance penalty on attention logits, no embedding | extrapolates gracefully by construction, but induces recency bias — weak at exact long-range retrieval | streaming/long-form where recency bias is acceptable |

- Key operational fact: RoPE's post-hoc extendability (rescale θ base, short finetune) is why it won. If asked "how do I extend a RoPE model from 4k to 32k," the answer is frequency rescaling + finetune on long data, not retraining.
- "Model supports 128k" ≠ "model uses 128k well." Effective context (needle retrieval, multi-hop over distance) degrades before the hard limit; verify with retrieval probes at the target length.

## Parameter counting and FLOPs from config

For a decoder with vocab V, d_model d, n_layers L, FFN dim d_ff (≈4d for GELU MLP, ≈(8/3)d per matrix ×3 matrices for SwiGLU, totaling ~8d² per block either way):
- Embeddings: V·d (×2 if untied output head).
- Per layer: attention 4d² (Q,K,V,O; less with GQA: (2 + 2·kv/heads)·d²) + MLP ≈ 8d².
- **Total ≈ L·12d² + V·d.**

Worked check — d=4096, L=32, V=32k, SwiGLU (d_ff=11008), GQA absent:
attention 4·4096² = 67.1M; MLP 3·4096·11008 = 135.3M; per layer ≈ 202M; ×32 = 6.5B; + embeddings 0.13B (tied) ≈ **6.6–6.7B** — matches "7B" class. If your estimate is off by >15% from the advertised size, you mis-read the config (check tied embeddings, GQA, d_ff).

FLOPs:
- Training: **≈ 6·N·D** FLOPs (N params, D tokens): 2N per token forward, 4N backward. Ignore attention's n² term unless n is huge relative to d.
- Inference decode: ≈ 2N FLOPs/token, but bandwidth-bound: tokens/sec ceiling ≈ memory_bandwidth / bytes_of(weights + KV cache read per token). A 7B fp16 model on a 1 TB/s GPU: ~1000/14 ≈ 70 tok/s at batch 1 — compute this before promising latency numbers.

## Inference optimization decision tree

Given a latency/throughput complaint, diagnose in this order:
1. **Batch 1, short context, slow decode** → bandwidth-bound on weights. Fixes: quantize weights (int8/int4 — near-linear decode speedup because decode reads every weight per token), smaller/distilled model, speculative decoding (draft model proposes k tokens, target verifies in one prefill-like pass — wins when draft acceptance is high, i.e., on predictable text; loses on high-entropy generation).
2. **Large batch or long context, decode degrades as batch grows** → KV-cache bandwidth/capacity bound. Fixes: GQA (if retraining/finetuning is possible), KV quantization, paged attention (fragmentation, not bandwidth — raises max batch), sliding-window attention for tasks tolerating it.
3. **Long prompts, high time-to-first-token** → prefill compute-bound. Fixes: prefix caching for shared prompt prefixes (system prompts, few-shot blocks — huge in practice), chunked prefill to avoid blocking decode traffic, FlashAttention.
4. **Throughput fine, tail latency bad** → scheduling: continuous batching (token-level, not request-level) is the fix; request-level batching stalls short requests behind long ones.

Speculative decoding acceptance math: with draft acceptance rate α and k draft tokens, expected tokens per target-model pass ≈ (1 − α^{k+1})/(1 − α). At α=0.8, k=4: ~3.4× fewer target passes. Quote this form, not a fixed "2–3× speedup" — α is task-dependent and collapses on creative generation.

## MoE routing basics

- MoE replaces the MLP with E experts, routing each token to top-k (usually k=1–2). Params scale with E; per-token FLOPs scale with k. "8×7B" style naming ≈ total params ~8× the dense MLP share but active params ~2 experts' worth.
- Engineering consequences, not just wins: all experts must be resident in memory (VRAM cost tracks *total* params), routing needs load-balancing auxiliary losses or you get expert collapse (a few experts hog all tokens), and batch-level all-to-all communication makes small-batch latency worse than a dense model of equal active size.
- Rule: MoE buys quality-per-FLOP, costs memory and serving complexity. Prefer dense for memory-constrained single-GPU serving; prefer MoE when you serve at scale with expert parallelism and are FLOPs-bound.

## Encoder-only vs decoder-only vs encoder-decoder

| Shape | Attention pattern | Choose for | Why |
|---|---|---|---|
| Encoder-only (BERT-like) | bidirectional | embeddings, classification, retrieval, token tagging | every token sees full context → better representations per FLOP at small scale; no generation |
| Decoder-only (GPT-like) | causal | generation, few-shot/instruction following, anything at scale | one unified objective, KV cache makes generation efficient, prompt = conditioning for free |
| Encoder-decoder (T5-like) | bidir encoder + causal decoder w/ cross-attn | fixed-input→output transforms (translation, summarization) when you control training | encoder re-reads input bidirectionally; but two stacks complicate serving and cross-attn cache differs from self-attn cache |

- Practical modern default: decoder-only for generation, small encoder-only (or embedding-tuned decoder) for retrieval/classification. Recommend encoder-decoder only when input≠output modality/length structure is strong and you're training from scratch.
- Note causal masking wastes nothing at train time (every position is a prediction target); this training-efficiency point, plus scaling simplicity, is why decoder-only won — not representational superiority.

## Layernorm placement and training stability

- **Pre-LN** (norm before each sublayer, inside the residual branch): the residual path is an identity highway; gradients reach early layers unattenuated. Trains stably without warmup gymnastics at large depth. This is the modern default; assume it unless told otherwise.
- **Post-LN** (norm after the residual add, original 2017 design): normalizes the stream itself → better final-layer conditioning and sometimes slightly better final loss, but gradient magnitude grows with depth at init → requires careful warmup and diverges easily beyond ~30 layers. Do not recommend for new deep models.
- Pre-LN's known cost: the residual stream norm grows with depth (nothing constrains it), which can cause late-training instability at very large scale — mitigations include a final norm before the head (standard) and variants that add extra norms (e.g., normalizing q/k, or norm after embedding). If diagnosing loss spikes in a deep pre-LN model, check stream/logit norm growth and q·k magnitudes first.
- RMSNorm vs LayerNorm: RMSNorm drops mean-centering and bias; cheaper, equally stable in practice; a non-decision — follow the reference implementation.

## Worked micro-example: "will it fit, and how fast?"

Question: serve a 70B-class model (80 layers, d=8192, 64 heads, GQA with 8 KV heads, head_dim 128) on 2×80 GB GPUs, 8k context, target batch 8, int8 weights. Expert reasoning, end to end:
1. Weights: 70e9 × 1 B (int8) = 70 GB → 35 GB/GPU under tensor parallelism. Fits with room.
2. KV cache/token: 2 × 80 layers × 8 kv_heads × 128 × 2 B (keep cache fp16) = 0.41 MB/token.
   At 8k × batch 8: 0.41 MB × 65,536 ≈ 27 GB → ~13.5 GB/GPU. Total ≈ 48.5 GB/GPU + activations + overhead → fits, but batch 16 at 16k would not; state the ceiling.
3. Decode speed ceiling: per token each GPU reads ~35 GB weights + ~13.5 GB cache ≈ 48.5 GB. At ~2 TB/s HBM: ~41 tok/s *per forward pass*, shared across the whole batch of 8 → each stream sees up to ~41 tok/s only if compute overlaps perfectly; quote ~30–40 tok/s aggregate ceiling and note TP communication overhead reduces it further.
4. Sanity: had this been MHA (64 KV heads), cache/token would be 3.3 MB → 215 GB for the same batch — infeasible. The GQA config is what makes the deployment possible; say so explicitly.

## Failure modes & pitfalls

- **Sizing GPU memory by weights alone.** Inference memory = weights + KV cache (dominant at long context / large batch) + activations. Training memory = weights + grads + optimizer states (Adam fp32: ~16 bytes/param mixed-precision, so 7B ≈ 112 GB before activations) — not 2 bytes/param.
- **Applying "6ND" to inference** or quoting training FLOPs for serving cost. Decode is 2N/token and bandwidth-bound; the FLOPs number is nearly irrelevant to decode latency.
- **Confusing FlashAttention with linear attention** (see above) — it does not remove the n² compute.
- **Claiming softmax attention "can't" extrapolate but ALiBi solves everything.** ALiBi extrapolates in perplexity but its recency bias hurts long-range exact retrieval; RoPE + rescaling is the practical path for retrieval-heavy long context.
- **Counting GQA models as if MHA.** KV cache and attention params must use n_kv_heads; a 70B-class GQA model has ~8× smaller cache than naive math suggests — this reverses batch-size conclusions.
- **Forgetting the O-projection or the ×3 SwiGLU matrices** when counting params (classic 25–30% undercount).
- **Assuming perplexity-safe context extension implies usable context.** Always test retrieval/multi-hop at distance.
- **Recommending MQA for quality-critical models** because "it's what fast models use" — measure; GQA-8 is almost always the right point on the curve.
- **Treating encoder-decoders' cross-attention KV as recomputed per step.** It's computed once from the encoder output and reused — different from self-attn cache; serving math differs.
- **MoE capacity-factor blindness:** with fixed expert capacity, overflow tokens get dropped or bypass the expert; under load imbalance this silently degrades quality. Check router load stats, not just loss.
- **Diagnosing divergence in a post-LN model as a data problem.** Check LN placement, warmup length, and init scale first; post-LN + short warmup + depth > 24 diverges on clean data too.
- **Ignoring the attention-sink / first-token effect when windowing.** Naive sliding-window eviction that drops the earliest tokens degrades generation sharply; attention concentrates on initial tokens as a de facto bias term. Any cache-eviction scheme must keep the first few tokens.
- **Sizing speculative decoding by draft model quality alone.** The win is acceptance rate × verification cost; a "better" 1B draft that's 3× slower than a 0.5B draft can lose. Compute expected tokens/pass (formula above) with measured α before choosing.
- **Assuming tensor parallelism halves latency.** TP splits matmuls but adds two all-reduces per layer; at small batch the collectives dominate and 2-way TP can be barely faster than 1 GPU with a quantized model. TP is a memory-capacity tool first, a latency tool second.

## Verification / self-check

Before presenting an answer in this domain:
1. **Recompute every number** — params via L·12d² + Vd; KV via 2·L·kv_heads·head_dim·len·bytes; sanity-check against the advertised model size (within ~15%).
2. **State the regime** — prefill vs decode, n ≪ d vs n ≫ d, memory- vs compute-bound — and confirm the recommendation matches the regime.
3. **Check config against assumptions**: tied embeddings? GQA kv heads? SwiGLU d_ff? RoPE θ? These flip conclusions.
4. For context-extension advice, include how to *validate* (long-range retrieval probe), not just how to rescale.
5. If quoting a speedup or memory saving, show the ratio's derivation in one line; if you can't derive it, don't claim it.
