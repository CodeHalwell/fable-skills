---
name: local-and-open-models
description: Selecting, sizing, licensing, running, and fine-tuning open-weight LLMs on local or self-hosted hardware — VRAM arithmetic (params × quant + KV cache), MoE offload, GGUF quant levels, llama.cpp/Ollama/LM Studio/vLLM selection, license reading (Apache-2.0 vs community licenses), local-vs-API decisions, and hybrid routing. Load when someone asks "can I run X on Y GPU", "which open model", "is Q4 good enough", or wants to replace an API model with a self-hosted one.
---

# Local and Open-Weight Models

## Core mental model

- **Everything is memory arithmetic.** "Can I run it" and "how fast" are both memory questions: weights + KV cache + compute buffers must fit somewhere, and decode speed is bounded by memory bandwidth divided by bytes-read-per-token. Do the arithmetic before recommending anything; never answer "a 24GB card runs 32B models" without computing the KV cache for the user's actual context length.
- **MoE broke the old sizing rules.** A mixture-of-experts model has two parameter counts: total (what must be resident in RAM+VRAM) and active (what is read per token, which sets speed). A 117B-total/5.1B-active model can decode faster than a 14B dense model while needing 60+ GB of memory. Most flagship open models as of 2026 are MoE (DeepSeek V4, Qwen3.5, Kimi K2, GLM, gpt-oss, Mistral Large 3); reasoning that assumes dense scaling is now usually wrong.
- **"Open" is a gradient, not a binary.** Apache-2.0/MIT → modified-MIT with attribution triggers → community licenses with MAU caps and naming rules → research-only → weights-available-with-no-real-license. The license is a load-bearing engineering input: it decides whether you can ship, redistribute a fine-tune, or serve customers. Read the actual license file, not the repo's "open source" headline.
- **The capability gap to frontier API models is real but task-dependent.** As of 2026, top open MoE models are competitive on coding, extraction, summarization, and agentic tool use; the gap remains widest on long-horizon reasoning, obscure knowledge, and polish under adversarial inputs. "Local can do it" is a claim to verify per task with an eval, never to assert from a leaderboard.
- **Never swap models on vibes.** A model change (or even a quant change) is a deployment like any other: it requires a regression suite of your real prompts, run before and after, including tool-call format, JSON validity, and refusal behavior. Leaderboard deltas do not transfer to your task distribution.
- **Local buys control, not just privacy.** Grammar-constrained decoding (GBNF/JSON-schema in llama.cpp, guided decoding in vLLM), full logprobs, deterministic pinned versions, no deprecation on someone else's schedule. When a design needs logit-level control, local is sometimes the only option, not the cheap option.

## The landscape, as of mid-2026 — and how to read a license

Fast-moving; verify names/versions before citing, but the shape (verified mid-2026):

| Family | Current class | License | Gotchas |
|---|---|---|---|
| Qwen 3 / 3.5 (Alibaba) | Qwen3.5 flagship 397B-A17B; mid-size MoE (122B-A10B, 35B-A3B) and dense down to <1B | Apache-2.0 | Cleanest big family; default candidate pool for most tasks |
| DeepSeek V3.x/V4, R1 | V4-Pro 1.6T-A49B; V4-Flash 284B-A13B (Apr 2026) | MIT | Frontier-class but too big for consumer hardware; V4-Flash is the self-hostable one |
| Kimi K2.x (Moonshot) | ~1T-total MoE line | Modified MIT | Attribution clause: >100M MAU or >$20M/mo revenue must display "Kimi K2" in the product UI |
| GLM-4.x/5 (Z.ai) | Large agentic MoE line | MIT (GLM-4.6; check newer) | Strong coding/agentic reputation |
| gpt-oss (OpenAI, Aug 2025) | 120b (117B-A5.1B), 20b (21B-A3.6B), shipped MXFP4-native | Apache-2.0 | 120b fits 80GB; 20b fits 16GB; reasoning-effort control built in |
| Llama 3.x/4 (Meta) | Llama 4 Scout/Maverick (MoE), Llama 3.3 70B dense | Llama Community License | 700M-MAU cutoff, "Built with Llama" attribution, derivative naming rules, AUP; Llama 4 multimodal excluded EU-domiciled entities at launch. No longer the default choice — as of 2026 Chinese labs hold most top open-weight slots |
| Gemma 3 / 4 (Google) | Gemma 4 (Apr 2026) incl. small MoE for local | Gemma 3: custom Terms of Use with updatable prohibited-use policy that flows down to derivatives; **Gemma 4: Apache-2.0** | The Gemma 3→4 license change is the kind of fact to re-verify, not assume across versions |
| Mistral 3 era | Ministral dense 3/8/14B; Mistral Small 4 (119B-A6B); Large 3 (675B-A41B) | Apache-2.0 for the Mistral 3 line; some earlier/specialty models under Mistral Research License (non-commercial) | Check per-model — the family mixes licenses |

**License-reading discipline** — read for four things, in order:
1. **License class.** Apache-2.0/MIT: done, standard obligations only. Anything named "Community License", "Terms of Use", or "Modified X": keep reading.
2. **Acceptable-use policy incorporated by reference.** Many licenses bind you to a separate AUP the vendor can update. If your domain is sensitive (legal, medical, security tooling), check the AUP names it.
3. **Triggers and caps.** MAU/revenue thresholds (Llama 700M MAU; Kimi 100M MAU/$20M-mo attribution), naming requirements for derivatives, attribution text.
4. **Output and derivative clauses.** Some licenses restrict using outputs to train competing models or impose flow-down terms on fine-tunes you redistribute. If you'll ship merged fine-tuned weights, this clause decides your product license.
Also distinguish license from provenance: "open weight" almost never means open training data or reproducibility. If the customer says "open source" and means OSI-approved, only Apache/MIT-class qualifies.

## Sizing arithmetic: will it fit, and how fast

**Fit (VRAM/RAM budget):**
```
weights_bytes ≈ params_total × bits_per_weight / 8      (Q4_K_M ≈ 4.85 bpw ≈ 0.61 bytes/param)
kv_bytes/token = 2 × n_layers × n_kv_heads × head_dim × bytes_per_elem   (2 = K and V; fp16 elem = 2 bytes)
total ≈ weights + kv_bytes/token × context_len + compute buffers (~0.5–2 GB) + everything else on the GPU (display, other apps)
```
Get `n_layers`, `n_kv_heads`, `head_dim` from the model's `config.json` — do not guess; GQA ratios vary widely and dominate KV size. Models with sliding-window or hybrid-local attention (Gemma, gpt-oss) have smaller KV than the formula; treat it as an upper bound.

**Speed (decode):** generation is memory-bandwidth-bound: `tokens/s upper bound ≈ bandwidth / bytes_of_active_weights_read_per_token`. This one line explains most local-inference phenomena:
- RTX 4090 (~1 TB/s) on a 20 GB Q4 dense model → ~50 t/s ceiling, ~35–40 real.
- Dual-channel DDR5 (~80–100 GB/s) on the same dense model → ~4–5 t/s. This is why full-CPU dense inference is painful.
- The same DDR5 on a 3B-active MoE (Q4 ≈ ~2 GB read/token) → ~40–50 t/s ceiling. This is why MoE + CPU/RAM offload works: park the expert FFNs (the bulk of the bytes, sparsely accessed) in system RAM (`llama.cpp --n-cpu-moe N`), keep attention + KV + dense layers on GPU.
- Apple Silicon sits between: M4 Max ~546 GB/s unified memory — good decode, but weak compute means slow prompt processing; long-context RAG on a Mac has painful time-to-first-token even when decode t/s looks great. Also, macOS caps GPU-usable unified memory (~75% by default): a "128GB Mac" is not 128GB of model budget.
- Quote t/s at realistic context depth: decode slows as the KV cache grows (more bytes read per token). Empty-context benchmarks flatter every setup.

**MoE quality heuristic** (folklore, roughly matches evals): effective dense-equivalent ≈ √(total × active). Qwen3-30B-A3B ≈ √(30×3.3) ≈ 10B-dense quality at 3B-dense speed. Use it for triage, then eval.

## Choosing a model: the reasoning chain

Ask in this order:
1. **What's the hard requirement — capability, memory, latency, or license?** Fix the binding constraint first. If the license must be Apache/MIT for redistribution, that eliminates families before you look at any benchmark.
2. **What capability class does the task need?** Extraction/classification/format-following: small dense (4–14B) or small MoE often suffices. Coding assistant/agentic tool use: current-generation mid MoE (gpt-oss-120b, Qwen3.5-122B-A10B class) or bigger. Frontier reasoning: probably API — say so.
3. **Now the arithmetic:** for each candidate, compute weights-at-Q4 + KV at the user's real context. Prefer a bigger model at Q4 over a smaller one at Q8/FP16 at equal memory — measured evals consistently favor parameters over precision down to ~4 bpw.
4. **Context length is a budget line, not a footnote.** 32k of fp16 KV on a dense 32B model is gigabytes (worked example below). If the use case is long-document RAG, KV may dominate and push you to a GQA-heavy or sliding-window model, KV quantization (q8_0 is near-lossless), or a smaller model.
5. **Prefer boring, well-supported checkpoints.** A model with day-one llama.cpp/vLLM support, official GGUFs, and a large user base surfaces its bugs in someone else's deployment. Exotic architectures spend weeks with broken chat templates and misconverted quants.
What changes the answer: measured failure on your eval (move up a class), t/s below the UX floor (move down or MoE), license conflict (change family), context blowing memory (change architecture, not just size).

## Choosing a runtime: the reasoning chain

First question: **how many concurrent users, and who operates it?**
- **One user, wants zero friction** → Ollama. Wraps llama.cpp (MLX on Apple Silicon in recent versions, as of 2026); model pull/run UX; OpenAI-compatible endpoint. Cost: less control, and its defaults (notably small default context, see pitfalls) bite.
- **One user, GUI-first / non-shell team** → LM Studio. Best model browser, runs GGUF and MLX, headless daemon mode; free for commercial use as of 2025.
- **Maximum control, exotic/air-gapped/embedded hardware, or MoE CPU-offload tuning** → llama.cpp directly (`llama-server`). Widest backend list (CUDA, Metal, ROCm, Vulkan, SYCL, CPU), single binary, grammars, first place new quant types land. This is the engine under Ollama/LM Studio anyway; going direct removes an abstraction that hides flags you need.
- **Multi-user or production serving on real GPUs** → vLLM (or SGLang). Continuous batching + PagedAttention give 2–4×+ aggregate throughput once ~10 concurrent requests are in flight; at 1 user it's roughly a wash vs llama.cpp. Use FP8/AWQ/GPTQ checkpoints for vLLM — GGUF is llama.cpp-ecosystem; don't drag it into vLLM.
- **Apple-Silicon-only shop wanting max Mac performance** → MLX (directly or via LM Studio/Ollama backends).
All of these expose OpenAI-compatible endpoints, so clients are portable — pick the runtime per deployment, not per codebase. The classic error is Ollama in production for 40 users (requests queue serially per model by default) or vLLM on a laptop (it preallocates ~90% of VRAM by design).

## Quantization: folklore vs measurement

- The folklore — "Q4_K_M is the sweet spot, Q8 is overkill, below Q3 is lobotomy" — is directionally right and precisely wrong. Measured: Q4_K_M costs a few percent perplexity vs fp16, but perplexity understates damage on code, math, and multi-step tool use; and damage at fixed bpw shrinks as models grow (70B at Q4 degrades less than 7B at Q4; small models want Q5/Q6+).
- **Importance-matrix (imatrix) quants are strictly preferable** when available: calibration data weights which tensors keep precision; the gain is largest at ≤4 bpw. Reputable quantizers (bartowski, unsloth) publish imatrix quants by default; a bare non-imatrix Q3/IQ2 from an unknown uploader is a different (worse) artifact with the same name.
- IQ-series quants (IQ4_XS, IQ3_M, …) beat legacy K-quants at low bpw at some CPU-speed cost; use them when squeezing a model onto a too-small GPU where all layers still fit.
- **KV-cache quantization** is a separate knob: `--cache-type-k q8_0 --cache-type-v q8_0` roughly halves KV memory near-losslessly; q4 KV (especially V) measurably hurts long-context recall — test before shipping.
- Decision rule: Q4_K_M (imatrix) as default for ≥14B; Q5_K_M/Q6_K for ≤8B or precision-sensitive tasks (code, math, function calling); Q8_0 only when memory is free; sub-4-bpw only to make an otherwise-impossible model possible, with eyes open and an eval. Whatever you choose: **a quant level change is a model change — run the regression suite.**
- Provenance hygiene: verify the GGUF is converted from the official base (repo lineage, tensor count sanity, the embedded chat template), not a random "improved" merge with the same name.

## QLoRA on local hardware: the arithmetic only

(Method, data, and hyperparameters live in the fine-tuning-and-adaptation skill; this is the will-it-fit math.)
- Unsloth's published minimums (as of 2026, batch size 1–2, their optimizations): QLoRA 4-bit ≈ **7B→5 GB, 14B→8.5 GB, 32B→26 GB, 70B→41 GB**; LoRA 16-bit ≈ 3× more (8B→22 GB, 70B→164 GB). Standard PEFT/TRL without Unsloth needs roughly 1.3–1.5× these. So: a 24 GB card comfortably QLoRA-tunes up to ~14B, marginally 27B, not 32B; 32B wants 32 GB (RTX 5090 class) or a rented A100/H100.
- Inference size ≠ training size: training adds gradients + optimizer state for the LoRA params (small) but chiefly **activations, which scale with sequence length × batch**. A run that fits at 2k sequence length OOMs at 16k; gradient checkpointing trades ~20–30% speed for most of that memory back. When someone reports OOM, ask for their max_seq_len and batch size before their model size.
- MoE fine-tuning locally is mostly a trap as of 2026: expert weights dominate memory and tooling support is uneven — for local budgets, tune a dense model (Qwen dense lines, Ministral, gemma-class) or gpt-oss-20b, or rent hours for anything bigger. Renting an H100 for a weekend is usually cheaper than the hardware conversation.

## Local vs API vs hybrid: the honest framework

Local wins on: **data residency/compliance** (the only non-negotiable one), **offline/edge**, **marginal cost at high sustained utilization**, **latency floor** (no network; and constrained decoding avoids retry loops), **control** (pinned versions, logprobs, grammars, no deprecations). API wins on: **frontier capability**, **zero ops** (no CUDA drivers, no on-call, no capacity planning), **burst elasticity**, **cost at low/spiky utilization**.

The two dishonest arguments to preempt:
- *Cost:* local is cheaper only when utilization is high. Amortize hardware + power + the engineer-hours of ops against tokens actually served; a GPU serving 2 hours/day loses to an API on TCO almost every time. Batch/offline workloads (nightly classification of a million records) are local's best economic case; a low-traffic chatbot is its worst.
- *Capability:* if the task needs frontier reasoning, a local model missing it doesn't save money — it fails cheaply. Run the eval; report the gap; let the owner decide with numbers.

**Hybrid patterns that work:** (1) local-first with API escalation — local handles the 80–95% of routine traffic, a router escalates hard cases; route on measured task difficulty or local-model self-consistency/confidence, never on query length. (2) Privacy-partitioned — PII-bearing steps run local by policy, anonymized/aggregate steps may use API; the escalation path must respect the partition (see pitfalls). (3) Local as fallback for API outages — cheap insurance, but only if you run the regression suite against the fallback too, or your "fallback" is an unshipped model.

## How an expert thinks through this

*Scenario: "We want a self-hosted coding assistant for our 40-engineer team — code can't leave the building. We have budget for one GPU server. Which model, which stack?"*

Constraint first: code can't leave → local is justified by policy, no cost debate needed. Next binding constraint: 40 concurrent-ish users → this is a serving problem, so vLLM-class runtime on datacenter GPUs; Ollama is out immediately (single-user queueing semantics), llama.cpp-server possible but batching under concurrency is vLLM's home turf.

Capability class: agentic coding is the hardest common local task — small dense models frustrate developers and get abandoned. Candidate pool as of 2026: gpt-oss-120b (Apache-2.0, 117B-A5.1B, fits ~61 GB in native MXFP4), Qwen3.5 mid-size MoE (Apache-2.0), GLM-4.x (MIT, strong coding, but 350B+ total needs multi-GPU). Consider and reject: **Llama 3.3 70B dense** — license workable but it's 2024-generation quality and 70B dense is slower per token than a 5–10B-active MoE that beats it on evals; dense-70B thinking is stale. Reject **DeepSeek V4-Pro** — frontier quality but 1.6T total params is a cluster, not a server. Reject **Kimi K2-class** for the same memory reason (also note its attribution trigger is irrelevant at 40 users, so license wasn't the objection — memory was).

Hardware sizing for gpt-oss-120b: weights ~61 GB → one H100 80 GB or two 48 GB cards (RTX 6000 Ada / L40S) with tensor parallelism. Then the step most people skip: **concurrency KV budget.** 40 engineers, say 8 active requests at 16–32k context — KV for MoE models is per-layer attention as usual; with ~19 GB free on an 80 GB card after weights, check vLLM's KV-page math (`--max-model-len`, watch its startup log line reporting KV capacity) rather than hand-waving. If it doesn't fit, options in order: KV fp8, cap max context, second GPU. Prompt processing (big pastes of code) is compute-heavy — datacenter cards fine, would have been the hidden killer on a Mac Studio "server" (rejected: great decode, weak prefill; 40 users would stack up on time-to-first-token).

Before rollout: build the regression suite from real internal tasks (50–200 prompts: edits, reviews, tool-call transcripts), score gpt-oss-120b vs the incumbent API model, and verify tool-calling end-to-end in the actual IDE plugin — tool-call parsing (`--tool-call-parser`) breaks silently across model families. If the eval shows a material gap on the hardest tasks, propose hybrid: local by default, opt-in API escalation with a code-redaction gate — and let security own that decision. Stopping rule: when the local model wins or ties on ≥90% of suite tasks and the remainder have a routing story, ship; don't chase the last benchmark point.

## Failure modes & pitfalls

- **Sizing by weights file alone.** "The GGUF is 20 GB, I have 24 GB, it fits" — then OOM (or silent slow CPU spill) at 32k context because KV + compute buffers + desktop compositor need 6–10 GB more. Always compute KV from `config.json` at the real context length; on shared desktops subtract 1–2 GB for the display.
- **Ollama's silent context truncation.** Ollama defaults to a small context window (historically 2k–4k via `num_ctx`/`OLLAMA_CONTEXT_LENGTH`) regardless of what the model supports. Symptom: "the model ignores the middle of my document" / RAG answers degrade — the prompt was truncated without error. First question for any Ollama-based RAG bug: what is `num_ctx`?
- **Judging a quant (or model) swap by chat vibes.** Q4 vs Q5 differences hide in code correctness, arithmetic, and tool-call JSON — exactly what casual chat doesn't exercise. Ship gate: fixed regression suite, before and after, including structured-output validity rate and refusal probes.
- **Perplexity as the only quant metric.** ΔPPL of "only 3%" can coexist with a visible drop in pass@1 on code. Perplexity ranks quants of the same model; it doesn't certify task fitness. Task evals or it didn't happen.
- **MoE total/active confusion, both directions.** (a) "30B-A3B only needs 3B of memory" — no: all experts must be resident (VRAM or RAM); active count sets speed, total sets footprint. (b) "I can't run 30B on a 12 GB GPU" — yes you can: `--n-cpu-moe` parks expert FFNs in system RAM; only ~2 GB/token of experts stream from RAM, so it's fast. Missing (b) makes people buy hardware they don't need.
- **Chat-template drift.** GGUFs embed a chat template; runtimes sometimes override or mis-render it (llama.cpp needs `--jinja` for many modern templates; a raw `/completion` endpoint bypasses templating entirely). Symptom: model is "dumber locally than on the hosted demo," emits role tokens, or never stops. Diff the fully rendered prompt against the model card's template before blaming the quant.
- **License read at the headline level.** Treating Llama's community license as Apache (it has MAU caps, naming rules for derivatives, attribution, an AUP); missing that Gemma 3's terms flow down to fine-tunes and its prohibited-use policy is updatable by Google (Gemma 4 moved to Apache-2.0 — version matters); missing Mistral Research License (non-commercial) on some Mistral checkpoints in an otherwise Apache family. Also the quiet clause class: restrictions on using outputs to train competing models. Five minutes reading the LICENSE file beats a legal escalation later.
- **vLLM on the wrong box / wrong mental model.** vLLM preallocates `--gpu-memory-utilization` (default 0.9) of VRAM at startup for weights+KV pages — it's not "using" that for the model, and it will fight your desktop session or a second process. Conversely, complaining llama.cpp "only" does 40 t/s aggregate for 20 users is using a single-stream engine for a batching job.
- **Benchmarking t/s at zero context.** Decode slows as KV grows; a "45 t/s" model doing 22 t/s at 20k-token depth is expected physics, not a regression. Benchmark at the context depth the workload actually runs at, and report prefill (prompt processing) separately — on Macs and iGPUs, prefill is the bottleneck users actually feel.
- **Mac unified-memory optimism.** 64 GB unified ≠ 64 GB model budget: macOS caps GPU wired memory (~75% default), the OS needs several GB, and prefill compute is far below CUDA-class. Macs are superb single-user decode machines and poor long-context/multi-user servers. Size against the usable fraction and test time-to-first-token with real prompts.
- **Random GGUF roulette.** Downloading whichever quant tops HF search: unlabeled non-imatrix quants, "uncensored/improved" merges masquerading as the base model, broken template metadata, or quants of a stale base revision. Prefer the model creator's official GGUFs or high-reputation quantizers; check the lineage on the model card.
- **Tool-calling assumed portable across models.** Function-call wire formats differ (Hermes-style, native formats, XML-ish variants); the server must parse the model's specific format (vLLM `--tool-call-parser`, llama.cpp template-dependent). A model swap that "works in chat" can zero out your agent's tool-call success rate. The regression suite must include real tool-call transcripts.
- **Hybrid router that defeats the point.** Local-first "for privacy" with automatic API fallback that ships the same PII upstream on hard queries — the architecture leaks exactly the data it was built to protect, on exactly the trickiest inputs. Escalation paths need redaction or an explicit consent/policy gate, and difficulty-routing must not use raw content in third-party routing services either.
- **Cost case built on peak utilization.** "One H100 replaces $8k/month of API" assumes the GPU runs hot 24/7. Idle GPUs still cost capital, power, and the engineer who patches CUDA drivers. Do TCO at measured utilization; recommend batch/offline consolidation to raise it, or concede the API is cheaper at that volume.
- **QLoRA OOM misdiagnosis.** The model "fits" per the sizing table but training OOMs — because activations scale with sequence length × batch, not model size. Fix order: batch→1 with gradient accumulation, enable gradient checkpointing, cut max_seq_len to the data's real p95; only then shrink the model. Also: the sizing tables are optimized-stack minimums (Unsloth-class); vanilla TRL needs ~1.3–1.5×.
- **Fine-tuned-then-quantized artifact never re-evaled.** Merging a LoRA into fp16 and re-quantizing to GGUF shifts numerics twice; the deployed artifact is not the model you evaluated at train time. Eval the exact GGUF in the exact runtime you ship.
- **Sampler defaults differ per runtime and silently change the model.** The same GGUF behaves differently under Ollama, llama.cpp, and LM Studio because temperature, top_p, min_p, and especially **repeat penalty** defaults differ. A repeat penalty >1.0 (a common local default) actively damages code generation and structured output — it penalizes the repeated braces, indentation, and field names that correct code requires. For code/JSON workloads set repeat penalty to 1.0 explicitly, and pin the model card's recommended sampler settings (Qwen and DeepSeek publish them; deviating is a known quality cliff for reasoning variants). When comparing runtimes, normalize samplers first or you're comparing settings, not engines.
- **NVIDIA sysmem fallback masking OOM as slowness.** On Windows, the driver's "CUDA sysmem fallback" spills over-committed VRAM into system RAM instead of erroring: the model loads "fine" and runs 10× slower. A local model that got mysteriously slow after a context increase usually didn't regress — it started swapping. Check actual VRAM residency (nvidia-smi) rather than trusting a successful load; disable the fallback in the driver settings for predictable OOMs.
- **Day-one GGUFs of new architectures.** Quants uploaded before (or with) llama.cpp support for a new architecture are frequently mis-converted: wrong RoPE scaling, wrong tokenizer pre-processing, missing template — producing subtly degraded or outright broken output that gets blamed on the model. If a model released this week looks bad locally, suspect the conversion and runtime version before the weights; re-check after the quantizer re-uploads (reputable ones re-convert after early fixes) and pin the llama.cpp build that officially added the arch.
- **Reasoning models blowing the token budget you sized for.** R1/gpt-oss-class models emit thinking traces that can be several thousand tokens per turn: KV and latency budgets sized for answer-length outputs are wrong by 5–20×, and multi-turn agents must strip prior thinking from history (correct chat templates do this — verify yours does) or the context fills with stale reasoning. Budget max output tokens, use effort/thinking controls where the model exposes them (gpt-oss reasoning effort; hybrid think/no-think modes in Qwen-class models), and measure cost per *task*, not per token.

## Worked micro-examples

**1. "Will Qwen3-32B fit my RTX 4090 (24 GB) at 32k context?"** (config verified: 64 layers, 8 KV heads, head_dim 128, 32.8B params)
```
Weights, Q4_K_M:  32.8e9 × 4.85/8 bits ≈ 19.9 GB   (matches the ~19.8 GB published GGUF)
KV per token fp16: 2 × 64 × 8 × 128 × 2 B = 262,144 B ≈ 0.25 MB
KV at 32k:        0.25 MB × 32,768 ≈ 8.0 GB
Total:            19.9 + 8.0 + ~1.5 (buffers) ≈ 29.4 GB → does NOT fit in 24 GB.
Options, in preference order:
  a) KV q8_0 (near-lossless): 19.9 + 4.0 + 1.5 ≈ 25.4 GB → still no.
  b) IQ4_XS weights (~4.3 bpw ≈ 17.6 GB) + KV q8_0 ≈ 23.1 GB → marginal; headless box only.
  c) Cap context at 16k with KV q8_0: 19.9 + 2.0 + 1.5 ≈ 23.4 GB → workable if 16k suffices.
  d) Switch architecture: Qwen3-30B-A3B (MoE) — similar quality class, 3.3B active → faster,
     and experts can spill to RAM. Usually the right answer.
Speed sanity check (option c): ~1 TB/s ÷ 19.9 GB ≈ 50 t/s ceiling → expect ~35 t/s real.
```
The headline "a 4090 runs 32B" is true only at short context — the KV line is where the folklore breaks.

**2. Big MoE on a small GPU (the 2026 move):** gpt-oss-120b (117B total, 5.1B active, ~61 GB MXFP4) on a 16 GB GPU + 64 GB DDR5:
```
llama-server -m gpt-oss-120b-mxfp4.gguf -ngl 99 --n-cpu-moe 28 -c 16384
```
`-ngl 99` puts all layers nominally on GPU; `--n-cpu-moe 28` overrides expert tensors of 28 layers back to CPU RAM. Attention + KV + dense layers stay on the GPU; ~2–3 GB of active expert weights stream from RAM per token → reported ~20–30 t/s class on DDR5 desktops, versus ~2 t/s if you'd naively split whole layers (`-ngl 20`) and forced attention through system RAM. Tune `--n-cpu-moe` downward until VRAM is nearly full — every expert layer moved back to VRAM is free speed.

**3. Model-swap regression harness (the gate for every swap — model, quant, or runtime):**
```python
# suite.jsonl: 50-200 real prompts with checkers, mined from production logs:
# {"prompt": [...messages...], "check": "json_schema"|"contains"|"tool_call"|"judge", "arg": ...}
import json, requests
def run(base_url, model, case):
    r = requests.post(f"{base_url}/v1/chat/completions", json={
        "model": model, "messages": case["prompt"],
        "temperature": 0, "max_tokens": 2048,          # pin samplers — defaults differ per runtime
    }).json()["choices"][0]
    out, calls = r["message"].get("content") or "", r["message"].get("tool_calls") or []
    if case["check"] == "json_schema":  return validates(out, case["arg"])
    if case["check"] == "tool_call":    return calls and calls[0]["function"]["name"] == case["arg"]
    if case["check"] == "contains":     return case["arg"] in out
    if case["check"] == "judge":        return llm_judge(case, out)   # strong model, fixed rubric

for name, url, model in [("incumbent", API_URL, "current-model"),
                         ("candidate", "http://localhost:8000", "local-gguf")]:
    scores = [run(url, model, json.loads(l)) for l in open("suite.jsonl")]
    print(name, sum(scores) / len(scores))
```
Everything speaks OpenAI-compatible HTTP, so the identical harness scores an API incumbent, a vLLM candidate, and a llama.cpp quant — which is the point: one fixed suite, per-category scores (code, extraction, tool calls, refusals), swap only when the candidate ties or wins per category, not just on the average.

**4. QLoRA budget check:** "Can I fine-tune Qwen3-14B on my 4090 for a format-adaptation task?" Sizing table: 14B QLoRA ≈ 8.5 GB minimum → yes, with room for seq_len 4–8k at batch 2 + gradient checkpointing on 24 GB. Same question for 32B: ≈26 GB minimum → no on 24 GB; answer is a 32 GB card, a rented A100 (~$1–2/hr spot gets the whole run under $50), or reframing to 14B — not heroic memory tricks that consume a week.

## Verification / self-check

Before presenting a recommendation in this domain, confirm:
- [ ] You computed, not asserted, the memory: weights bpw × params + KV(config.json, real context) + buffers — and stated the context length the answer assumes.
- [ ] Every named model/version/license was verified current (this space turns over in months; anything you "remember" about model generations or license terms may be a generation stale — mark facts "as of 2026" and prefer stable principles over version trivia).
- [ ] For MoE: you used total params for fit, active params for speed, and considered expert offload before declaring hardware insufficient.
- [ ] The license answer came from the license text class (Apache/MIT vs community/custom), including AUP, MAU/attribution triggers, and derivative clauses — not from the word "open" in a blog post.
- [ ] Runtime matches deployment shape: single-user convenience (Ollama/LM Studio) vs control (llama.cpp) vs concurrent serving (vLLM/SGLang) — and you didn't recommend GGUF into vLLM or Ollama into multi-user production.
- [ ] Any model/quant swap recommendation includes the regression-suite step (task metrics + tool-call/JSON validity + refusals) as a gate, not a suggestion.
- [ ] The local-vs-API answer states the capability gap honestly and does the cost math at realistic utilization; if the true answer is "use the API for this," you said so.
- [ ] Stopping rule: fit arithmetic done, license class confirmed, eval gate defined → stop. Further micro-optimization of quant levels or flags without an eval in hand is waste.
