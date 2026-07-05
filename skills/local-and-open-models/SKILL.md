---
name: local-and-open-models
description: Selecting, sizing, licensing, running, and fine-tuning open-weight LLMs on local or self-hosted hardware — VRAM arithmetic (params × quant + KV cache), MoE offload, GGUF quant levels, llama.cpp/Ollama/LM Studio/vLLM selection, license reading (Apache-2.0 vs community licenses), local-vs-API decisions, and hybrid routing. Load when someone asks "can I run X on Y GPU", "which open model", "is Q4 good enough", or wants to replace an API model with a self-hosted one.
---

# Local and Open-Weight Models

## The standard doctrine, compressed

A strong model already produces this cold; anchors only. Everything is memory arithmetic: weights (Q4_K_M ≈ 4.85 bpw) + KV from `config.json` (2 × layers × kv_heads × head_dim × bytes) at the *real* context + 1–2 GB buffers; decode tok/s ≤ bandwidth ÷ active-bytes/token (4090 ~1 TB/s → ~35–40 real on a 20 GB dense model; DDR5 ~80–100 GB/s → 4–5 on dense, but ~20–40 on a 3B-active MoE — which is why `--n-cpu-moe` expert offload works: experts stream from RAM, attention+KV stay on GPU). MoE: total params = footprint, active = speed, √(total×active) ≈ dense-equivalent quality. Worked example a strong model reproduces exactly: Qwen3-32B Q4 at 32k fp16 KV = 19.9 + 8.0 + 1.5 ≈ 29 GB → doesn't fit 24 GB; options: KV q8_0, IQ4_XS, 16k cap, or switch to a 30B-A3B MoE (usually right). Quant folklore calibrated: Q4_K_M fine ≥14B, small models want Q5/Q6+, damage shrinks with size, imatrix quants strictly preferable ≤4 bpw, KV q8_0 near-lossless / q4 V-cache hurts long-context recall, perplexity understates code/math/tool damage — task evals or it didn't happen. Runtime by concurrency: Ollama (single-user, beware silent `num_ctx` truncation — first question for any local RAG bug), LM Studio (GUI), llama.cpp direct (control, grammars, exotic backends), vLLM/SGLang for multi-user (preallocates ~90% VRAM by design; wants AWQ/GPTQ/FP8, not GGUF). Repeat penalty >1.0 breaks code/JSON (penalizes required braces/indentation) — set 1.0 and pin model-card samplers; sampler defaults differ per runtime, so normalize before comparing. Macs: great single-user decode, weak prefill (long-context TTFT pain), ~75% of unified memory GPU-usable. QLoRA minimums (Unsloth-class): 7B→5 GB, 14B→8.5 GB, 32B→26 GB, 70B→41 GB; vanilla TRL ~1.3–1.5×; OOM cause is activations (seq_len × batch), not model size — batch 1 + grad checkpointing + cut seq_len first. Windows NVIDIA sysmem fallback masks OOM as 10× slowness — check nvidia-smi residency, disable fallback for loud OOMs. Every model/quant/runtime swap gates on a fixed regression suite (code, extraction, tool-call format, JSON validity, refusals) — leaderboard deltas don't transfer.

## The landscape, as of mid-2026 — the part a model's training data gets wrong

A strong model's cold answer describes the **2025 landscape** (Qwen3-235B, DeepSeek-V3/R1, Llama 4 as headline, Gemma 3, Phi-4). Verified mid-2026 shape:

| Family | Current class | License | Gotchas |
|---|---|---|---|
| Qwen 3 / 3.5 (Alibaba) | Qwen3.5 flagship 397B-A17B; mid MoE (122B-A10B, 35B-A3B); dense to <1B | Apache-2.0 | Cleanest big family; default candidate pool |
| DeepSeek V3.x/V4, R1 | V4-Pro 1.6T-A49B; **V4-Flash 284B-A13B (Apr 2026)** — the self-hostable one | MIT | Frontier-class; V4-Pro is a cluster, not a server |
| Kimi K2.x (Moonshot) | ~1T-total MoE line | Modified MIT | **Attribution trigger: >100M MAU or >$20M/mo revenue must display "Kimi K2" in the UI** |
| GLM-4.x/5 (Z.ai) | Large agentic MoE | MIT (GLM-4.6; check newer) | Strong coding/agentic reputation |
| gpt-oss (OpenAI) | 120b (117B-A5.1B, ~61 GB MXFP4-native), 20b (fits 16 GB) | Apache-2.0 | Reasoning-effort control built in |
| Llama 3.x/4 (Meta) | Llama 4 Scout/Maverick (MoE), 3.3-70B dense | Community License (700M MAU, naming, AUP; Llama 4 excluded EU entities at launch) | **No longer the default** — Chinese labs hold most top open-weight slots as of 2026; 70B-dense thinking is stale (slower than a 5–10B-active MoE that beats it) |
| Gemma 3 / 4 (Google) | Gemma 4 (Apr 2026) incl. small MoE | Gemma 3: custom updatable-AUP terms flowing to derivatives; **Gemma 4: Apache-2.0** | The 3→4 license change is exactly the fact cold answers miss — models assert "Gemma was never Apache" |
| Mistral 3 era | Ministral 3/8/14B dense; Small 4 (119B-A6B); Large 3 (675B-A41B) | Apache-2.0 for the Mistral 3 line; some earlier/specialty under MRL non-commercial | Mixed-license family — check per model |

**License-reading discipline** (read the file, not the headline): 1) class — Apache/MIT done; "Community License"/"Terms of Use"/"Modified X" keep reading; 2) AUP incorporated by reference and vendor-updatable? (matters for legal/medical/security products); 3) triggers — MAU/revenue caps, derivative naming, attribution strings; 4) output/derivative clauses — training-competing-models restrictions and flow-down terms decide your product license if you ship merged fine-tuned weights. "Open weight" ≠ open data/reproducible; if the customer means OSI-approved, only Apache/MIT-class qualifies.

## Sharpenings

- **MoE fine-tuning locally is mostly a trap (2026):** expert weights dominate memory, tooling uneven — tune a dense model or gpt-oss-20b, or rent an H100 weekend (~$50 of spot beats the hardware conversation).
- **Reasoning models blow token budgets 5–20×:** thinking traces run thousands of tokens/turn; KV and latency budgets sized for answer-length outputs are wrong, and multi-turn agents must strip prior thinking from history (verify the chat template does). Use effort/think-mode controls; measure cost per *task*.
- **Day-one GGUFs of new architectures are frequently mis-converted** (RoPE scaling, tokenizer pre-processing, template) — if a model released this week looks bad locally, suspect the conversion and llama.cpp version before the weights; re-check after reputable quantizers re-upload. Same hygiene: prefer official/bartowski/unsloth imatrix quants; diff the rendered chat template (`--jinja`) against the model card before blaming a quant.
- **Tool-calling is not portable across model families:** wire formats differ (Hermes-style, native, XML-ish); the server must parse the model's format (vLLM `--tool-call-parser`). A swap that "works in chat" can zero the agent's tool-call success — regression suite must include real tool-call transcripts, scored per category (swap only when the candidate ties/wins per category, not on the average).
- **Hybrid routers that defeat their own point:** local-first "for privacy" with automatic API fallback ships the same PII upstream on exactly the hardest inputs. Escalation needs a redaction or consent gate; difficulty-routing must not send raw content to third-party routing services. And the local-fallback-for-API-outages pattern only works if the regression suite ran against the fallback too.
- **The 16 GB-GPU MoE move, verbatim:** `llama-server -m gpt-oss-120b-mxfp4.gguf -ngl 99 --n-cpu-moe 28 -c 16384` — tune `--n-cpu-moe` down until VRAM is nearly full; every expert layer moved back to VRAM is free speed. ~20–30 t/s class on DDR5 desktops vs ~2 t/s for a naive `-ngl 20` whole-layer split that forces attention through system RAM.
- **Cost honesty in both directions:** local wins on residency/offline/high-sustained-utilization/control (grammars, logprobs, pinned versions); a GPU serving 2 h/day loses to the API on TCO nearly always; and a local model missing frontier capability doesn't save money — it fails cheaply. Run the eval, report the gap, let the owner decide with numbers.

## Verification / self-check

- Memory computed, not asserted (weights bpw + KV from config at stated context + buffers); MoE fit on total, speed on active, offload considered before "buy hardware."
- Every named model/version/license verified current — this table rots in months; anything "remembered" about generations or license terms may be one generation stale.
- License answer from the license text class incl. AUP/triggers/derivative clauses.
- Runtime matches deployment shape; no GGUF-into-vLLM, no Ollama-into-multi-user-production.
- Swap recommendations include the regression-suite gate; local-vs-API states the capability gap and does TCO at measured utilization — if the answer is "use the API," say so.
- Stopping rule: fit arithmetic + license class + eval gate defined → stop; flag-tuning without an eval in hand is waste.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 11 baseline (cut/compressed — full sizing arithmetic incl. the Qwen3-32B worked example, --n-cpu-moe mechanics and why-MoE-offload-works, √(t×a) heuristic, quant/imatrix/KV-quant calibration, num_ctx trap, repeat-penalty-vs-code, Mac assessment with 75% cap, QLoRA table + activation-OOM cause, runtime selection, sysmem fallback incl. the driver setting, swap-pitfall inventory), 0 partial, 2 delta (expanded).
- Biggest baseline gaps: the mid-2026 model landscape is a full generation stale in cold answers (no Qwen3.5/DeepSeek-V4/Kimi-K2/GLM; Llama still framed as default); Gemma licensing answered wrong ("never Apache" — Gemma 4 moved to Apache-2.0 in Apr 2026). These two are the file's reason to exist.
- Kept as likely-delta despite unprobed: Kimi K2 attribution trigger numbers, reasoning-model token-budget blowout, day-one-GGUF misconversion prior, hybrid-router privacy leak.
