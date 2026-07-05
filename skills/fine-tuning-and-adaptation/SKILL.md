---
name: fine-tuning-and-adaptation
description: Load when deciding whether to fine-tune an LLM, designing SFT/LoRA/QLoRA/DPO training runs, curating fine-tuning data, choosing hyperparameters, or debugging a fine-tuned model that got worse (forgetting, format lock-in, overfitting). Also for "should we fine-tune or just prompt/RAG?" questions.
---

# Fine-Tuning and Model Adaptation

## Core mental model (anchors — you hold these views; don't trade them away in a review)

- Fine-tuning teaches form and behavior, not facts; "tune on our docs" yields fluent hallucination in house style → RAG.
- Ladder with evidence per rung: prompting → few-shot → RAG → fine-tune, climbing only when the current rung plateaus *on a fixed eval set*. Strongest economic cases: distillation of a big model onto a small one for one narrow task, and folding a mega-prompt into weights at volume.
- Data quality dominates hyperparameters; the prompted-base-model row is mandatory in every results table (it often scores within noise of the tune, for free).
- LoRA defaults: r=16, alpha=2r, all linear layers (attention-only q/v targeting is the legacy tutorial mistake and costs real quality), LR 1e-4–2e-4 (~10× full-FT), dropout 0.05 small data. QLoRA = NF4 + bf16 compute, quality within noise, ~1.5–2× slower per step; use only when memory-bound.
- Forgetting: detect with a general-capability + refusal probe suite before/after; mitigate by training less, mixing 10–30% general instruction data, LoRA over full FT.
- DPO: SFT first (DPO reshuffles mass among behaviors the model already has); beta 0.1; LR ~10× *lower* than SFT; rejected = plausible near-misses sampled from your own SFT model, never strawmen; watch length hacking.
- Template mismatch is the #1 silent killer: render train and serve prompts and diff *at the token-ID level*; EOS in the loss or the model never stops; ship the chat template with the artifact; re-eval the exact deployed artifact (merge→quantize→engine shifts numerics).

## The corrections (where the standard answer is incomplete)

**Label noise amplifies; it doesn't average out.** The intuition is that 10% flawed examples cost ~10% quality. Wrong direction: inconsistency is itself a signal — the model learns "this rule is optional" and produces the erroneous variant *more* than 10% of the time. This is why reading 100 random examples yourself and estimating the label-error rate is a gating step, not hygiene: every r% of noise is a floor on your error rate, not a dilution. Cheap filter that works: score every example with a strong LLM against a correctness rubric, drop the bottom 10–20%, spot-check the drops.

**Distillation lives or dies on the correctness filter, not the count.** Budget 5k–50k teacher outputs for narrow-task distillation, but the load-bearing step is filtering to *verified-correct* targets (ground truth match, code that runs, teacher-as-judge) — unfiltered teacher outputs distill the teacher's error rate at full fidelity into a model too small to recover. Same trap squared for self-training loops: each generation amplifies the previous model's biases.

**Few-shot success is the go signal, not the alternative's failure.** If 5–10 in-context examples move the metric a lot, the task is learnable from examples — that's *evidence fine-tuning will work well*. But run the arithmetic first: 20–50 in-context examples under prompt caching is often cheaper than a tune plus its maintenance tax (re-tune on every base-model upgrade, frozen model generation). 200 examples total → don't SFT yet; use them as few-shot pool + eval set and collect more.

**Diversity beats volume at fixed budget.** 1,000 examples covering 1,000 distinct input patterns outperform 10,000 covering 500. Dedup near-duplicates (embedding/MinHash) before counting your dataset — 50k examples that are 5k patterns × 10 paraphrases teach 5k things while your metrics claim 50k, and overweight those patterns.

**Train on the inference-time distribution, including the boring parts.** Same prompt template, same system prompt, same retrieved-context format if serving uses RAG, realistic input lengths, refusal/"not enough information" examples at a realistic rate (or the tune erases that behavior). Distribution mismatch between tuning data and serving traffic is the top cause of "trained great, serves badly" after template bugs.

## Compressed operational rules

- Epochs: 1–3 at ≥10k examples; 3–5 only ≤1k with per-epoch generation evals; "more epochs fixed it" = memorized it. Checkpoint per epoch; select by held-out *task metric* + general-probe non-regression, never train loss.
- Overfit tell for generation tasks: eval loss looks fine while quality collapses — sample on paraphrases of training inputs; verbatim regurgitation = memorized.
- Loss masked to completion tokens (`assistant_only_loss` in TRL / labels=-100 on prompt); effective batch 16–64 via accumulation.
- Format lock-in check: ask the tuned model a casual question — if it answers in training format, you over-trained or under-mixed.
- Serving: LoRA adapters (vLLM `--enable-lora`) for many tasks/tenants; merged weights for single-task simplicity; adapter + base version + data snapshot + eval report travel together, rollback = repoint adapter.
- Eyeball one fully rendered training sample (`tok.decode(train_dataset[0]["input_ids"])`) for template + EOS before every run; invisible characters (`\r\n`, NBSP, stray leading space) tokenize differently and get learned.

## Verification checklist before recommending or shipping a tune

- [ ] Ladder evidence on a fixed eval set; knowledge gaps routed to RAG.
- [ ] 100 examples human-read; dedup done; label-error rate estimated (it is your error floor).
- [ ] Token-level train/serve render diff clean; loss masking + EOS verified.
- [ ] Distillation targets correctness-filtered; refusal behavior present in data at realistic rate.
- [ ] Before/after: task eval, general probe, refusal probe, format-flexibility check — on the exact deployed artifact in the serving engine.
- [ ] Tune beats the best *prompted* baseline by more than the eval noise floor; otherwise ship the prompt.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus treats label noise as proportional dilution; misses the amplification mechanism (inconsistency teaches "rule is optional" → error rate exceeds noise rate).
  - Distillation sizing low (1k–10k, no filter emphasis); the correctness filter as the load-bearing step and self-training amplification absent.
  - "Few-shot works → fine-tune will work" inference and the in-context-examples-plus-caching-vs-tune cost comparison not surfaced; diversity-beats-volume quantification absent.
