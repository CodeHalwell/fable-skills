---
name: deep-learning-fundamentals
description: Load when debugging training dynamics or explaining/predicting neural-net behavior from math — gradient flow and backprop shapes, vanishing/exploding gradients, normalization and residuals, optimizer and loss-landscape reasoning, double descent and overparameterized generalization, softmax/cross-entropy gradients, or choosing among regularizers.
---

# Deep Learning Fundamentals: Math That Predicts Behavior

Assumed baseline (verified expert-grade cold; do not re-derive in answers): backprop shapes (dL/dW = xᵀg, db = g.sum(0), dx = gWᵀ); softmax+CE gradient = p−y and why CE beats MSE-on-softmax (saturating p(1−p) factor); Jacobian-product vanishing math and the fix ranking residuals > norm > init > clipping; lr/batch noise-scale scaling and its large-batch breakdown; double descent including epoch-wise; bf16 vs fp16 (range vs mantissa, loss scaling, fp32 accumulation); float64 central-difference gradcheck with ReLU-kink caveats; AdamW decoupling, lr×wd coupling, no decay on norms/biases; frozen-BN needing `.eval()` not just requires_grad; grad-accumulation ≠ big batch under BN/contrastive losses; global-valid-token loss normalization under DDP/accumulation; Chinchilla 20 tok/param vs deployment-optimal overtraining; µP for LR transfer across width.

## Discipline rules (the residual value of this skill)

- Any training-dynamics claim must name its observable (per-layer grad norms, update/weight ratio ≈1e-3, activation saturation fraction, dead-ReLU fraction). No observable = folklore.
- Any optimizer/regularizer comparison is invalid unless LR was swept per arm and schedules *completed* (mid-decay checkpoints understate final quality).
- Any overfit/generalize prediction must state the regime first (under- vs over-parameterized, label-noise level); never argue "params > data points → overfits."
- Diagnose NaNs in this order: loss numerics (log(0), 0-div) → LR/warmup → fp16 overflow → bad batch. Grad-norm spike on one step = data; steady ramp = dynamics.

## Symptom → cause (ordered checklist — check the top suspect with a 5-minute experiment before theorizing)

| Symptom | First suspects, in order |
|---|---|
| Loss flat at ln(C) from step 0 | LR=0/optimizer not stepping; frozen/detached params; labels shuffled independently of inputs |
| Loss ↓, val accuracy at chance | metric bug (argmax axis), label-mapping mismatch train-vs-val, wrong split |
| Repeated spike-recover | LR too high late (add decay); fp16 overflow; bad batches (inspect the spike-step batch) |
| Train ↓, val ↓, val *metric* flat | loss–metric mismatch; threshold/calibration, not learning |
| Great val, bad prod | group/temporal leakage; preprocessing mismatch; shift |
| First epoch great, then degrades | LR too high post-warmup; shuffle off; BN stats poisoning |
| Slow convergence, all "correct" | **double softmax** (plateaus at a suspiciously moderate value — trains, silently caps confidence); unnormalized inputs; LR 10–100× low; batch too small for BN |

## Sharpenings and less-common corrections (kept because they're where even strong answers thin out)

- **Catastrophic cancellation**: variance as E[x²]−E[x]² in low precision explodes for large-mean activations — Welford or subtract-mean-first. This exact issue is why hand-rolled LayerNorms diverge where `nn.LayerNorm` doesn't; check it before blaming the architecture.
- **"BatchNorm fixes internal covariate shift" is outdated** — the defensible mechanism is landscape smoothing + scale control. Also: BN < batch ~8 has garbage statistics (GroupNorm/LayerNorm); BN over variable-length sequences leaks across positions — that's *why* transformers use LayerNorm.
- **Train loss > val loss says nothing by itself** when dropout/augmentation are train-only — train is measured on a harder problem. Compare val metrics over time, not the raw gap.
- **Comparing a larger-batch run's final loss** without noting step count changed at fixed epochs: compare at equal tokens/samples seen and tuned LR, or the comparison is void.
- **Emergence claims**: exact-match metrics quantize smooth log-likelihood gains — before claiming a capability jump, check whether per-token loss on the task also jumps.
- **In fp32, a correct gradient shows rel_err ~1e-3 in a numerical check** — do not "fix" a correct implementation chasing fp32 noise; go to float64 where the pass bar is ~1e-6.
- Label smoothing breaks calibration-sensitive downstream use of logits and hurts distillation *from* the smoothed model — two costs usually omitted when it's recommended.
- Regularizer hierarchy discipline: augmentation adds information; everything else only restricts capacity. Modern LLMs train with dropout 0 — dropout is last resort for large models, not a default knob.

## Verification / self-check

1. Hand-derived gradients: shapes term-by-term, then one scalar vs autograd in float64 (tol ~1e-5).
2. Dynamics claims: observable named and logged.
3. Comparisons: LR swept per arm, schedules completed, ≥3 seeds or determinism mode for sub-1% claims (dataloader-worker seeding included — augmentation randomness alone exceeds many claimed gains).
4. Numerics: losses on logits; bf16 or loss-scaling; eval under `model.eval()` + `no_grad`.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found: none substantive — Opus produced every derivation, double descent (incl. epoch-wise), AdamW decoupling, masked-loss global normalization, and µP cold, often with more detail than the v2 skill.
- Skill restructured to discipline rules + symptom checklist + the handful of unprobed corrections (cancellation in custom norms, double-softmax signature, fp32 gradcheck noise) that justify remaining context cost.
