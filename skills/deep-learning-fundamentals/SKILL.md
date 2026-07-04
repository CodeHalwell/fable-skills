---
name: deep-learning-fundamentals
description: Load when debugging training dynamics or explaining/predicting neural-net behavior from math — gradient flow and backprop shapes, vanishing/exploding gradients, normalization and residuals, optimizer and loss-landscape reasoning, double descent and overparameterized generalization, softmax/cross-entropy gradients, or choosing among regularizers.
---

# Deep Learning Fundamentals: Math That Predicts Behavior

## Core mental model

- **Backprop is chain-rule bookkeeping with strict shape discipline.** Every gradient has exactly the shape of the thing it's the gradient of. dL/dW has W's shape; if your derivation produces anything else, you made an error. Shape-checking catches ~90% of hand-derivation mistakes before any math review does.
- **Training behavior is governed by the product of Jacobians through depth.** A gradient at layer 1 is a product of L layer-Jacobians. Products of matrices with spectral norm ≠ 1 shrink or blow up exponentially in L. Every trick — residuals, normalization, careful init, gradient clipping — exists to keep this product near identity.
- **SGD noise is a feature, not a bug.** Minibatch gradient noise scales like lr/batch_size and biases optimization toward flatter minima, which correlate with better generalization. This is why "just use full-batch with a huge LR" doesn't reproduce SGD results, and why LR and batch size trade off (roughly linearly, until it breaks).
- **In the overparameterized regime, classical bias-variance intuition inverts.** More parameters past the interpolation threshold typically *improves* test error (double descent). "Your model has more params than data points, it must overfit" is wrong as stated for modern nets; implicit regularization of (S)GD + architecture priors do real work.
- **Loss values lie; gradients and curvature tell the truth.** Diagnose training with gradient norms per layer, update-to-weight ratios (healthy: ~1e-3 per step), and activation statistics — not loss curves alone.

## Backprop with exact shapes (the reference derivation)

Linear layer y = xW + b, with x: (B, d_in), W: (d_in, d_out), upstream g = dL/dy: (B, d_out):
- **dL/dW = xᵀ g** — (d_in, B)(B, d_out) = (d_in, d_out) ✓ matches W.
- **dL/db = g.sum(axis=0)** — (d_out,) ✓. Forgetting the batch-sum is the classic bias-gradient bug.
- **dL/dx = g Wᵀ** — (B, d_out)(d_out, d_in) = (B, d_in) ✓.

Elementwise activation h = φ(z): dL/dz = dL/dh ⊙ φ'(z) — Hadamard, never a matmul. Writing it as a matmul (diag-Jacobian expanded) is technically correct but the implementation must be elementwise.

Softmax + cross-entropy (the elegant result). With logits z, p = softmax(z), one-hot target y, L = −Σ y log p:
- ∂p_i/∂z_j = p_i(δ_ij − p_j).
- dL/dz_j = −Σ_i (y_i/p_i)·p_i(δ_ij − p_j) = −y_j + p_j Σ_i y_i = **p − y**.
Consequences you should actually use: (1) the gradient is bounded in [−1,1] per logit — no gradient explosion from the loss itself; (2) it never saturates to exactly zero while wrong (unlike MSE-on-softmax, which has a p(1−p) factor that kills gradients at confident mistakes — this is *why* CE is the right loss for classification, state it when asked); (3) implementations must fuse log-softmax with NLL (`F.cross_entropy` on raw logits) — passing softmax outputs into `cross_entropy`, or taking log of softmax separately, causes silent numerical error and double-softmax bugs respectively.

## Why depth breaks gradients, and what actually fixes it

Mechanism: dL/dx₀ = (∏ₗ Jₗ) dL/dx_L. If typical singular values of Jₗ are s, gradient magnitude ~ s^L. s=0.9, L=50 → 0.005×; s=1.1 → 117×. Sigmoid/tanh saturation (φ' ≤ 0.25 for sigmoid) plus mis-scaled weights push s < 1 → vanishing; large init or no normalization pushes s > 1 → exploding.

Fixes ranked by mechanism, not folklore:
1. **Residual connections**: J = I + J_block. Products of (I + small) stay near I — gradients flow through the identity term regardless of block saturation. This is the single most important fix; it's why 100+-layer nets train at all.
2. **Normalization (BatchNorm/LayerNorm)**: re-centers and re-scales activations every layer so φ operates in its non-saturated range and Jacobian scale stays ~1 *throughout training*, not just at init. Also smooths the loss landscape (empirically reduces gradient Lipschitz constant), permitting larger LRs.
3. **Init (He for ReLU: Var = 2/fan_in; Xavier for tanh: 2/(fan_in+fan_out))**: sets s ≈ 1 at step 0 only. Necessary, insufficient — without norm/residuals it drifts.
4. **Gradient clipping**: a safety valve for spikes (rare bad batches, RNNs), not a fix for systematic explosion. If clipping activates every step, the LR or architecture is wrong.

Diagnostic rule: log per-layer grad norms. Vanishing shows as a monotone decay toward early layers; a healthy resnet shows roughly flat norms. Exploding shows first in the *last* layers' activations/logits growing over training.

## Loss landscape and optimizer intuition

- SGD's stationary distribution concentrates in minima whose flatness matches the noise scale ~ lr/batch. Practical consequences: (a) when you scale batch ×k, scale LR ×k to preserve noise (works until ~batch where curvature limits LR); (b) very large batch + short training generalizes worse without compensating (LR warmup, longer training); (c) sharp-minimum solutions found by full-batch methods often generalize worse at equal train loss.
- Adam vs SGD: Adam normalizes per-coordinate gradient scale — wins on transformers/NLP where gradient scales vary wildly across parameters (embeddings vs layernorm gains); plain SGD+momentum historically competitive on convnets. Default AdamW for anything transformer-shaped; don't burn time tuning SGD there.
- Warmup exists because early training has poorly conditioned curvature and Adam's second-moment estimates are noisy; loss spikes in the first ~1k steps → lengthen warmup before touching anything else.
- Learning-rate decay matters more than LR peak for final loss; "constant LR then sudden drop" and cosine end at similar places — but *evaluating mid-training at high LR underestimates* final quality. Don't compare checkpoints across different points of their decay schedules.

## Mixed precision and numerics (where math meets hardware)

- bf16 has fp32's exponent range but ~3 decimal digits of mantissa; fp16 has more mantissa but overflows at 65504. Consequences: fp16 needs loss scaling (gradients underflow) and overflows on large logits/attention scores; bf16 needs neither but its coarse mantissa makes *accumulation* the danger — always accumulate reductions (softmax denominators, norms, losses) in fp32. Default: bf16 autocast with fp32 master weights.
- Ops that must stay fp32 even under autocast: softmax/log-softmax, layernorm statistics, loss computation, large sums. PyTorch autocast handles the standard ones; custom kernels/losses must do it manually — a custom contrastive loss accumulating in bf16 gives silently wrong gradients at batch ≥ ~1k.
- Non-determinism ≠ bug: atomics in backward kernels (e.g., `scatter_add`, some attention backwards) make bitwise-identical reruns impossible on GPU without `torch.use_deterministic_algorithms(True)` (slower). Never chase a 0.1% metric difference between "identical" runs without first checking determinism settings and seed coverage (Python, NumPy, torch, CUDA, dataloader workers).
- Catastrophic cancellation: computing variance as E[x²]−E[x]² in low precision explodes for large-mean activations; Welford or subtract-mean-first. This exact issue is why naive custom layernorms diverge where `nn.LayerNorm` doesn't.

## Bias-variance in the overparameterized regime (double descent)

- Classical U-curve holds *below* the interpolation threshold (model can't fit train data). At the threshold, test error peaks — the model contorts to fit every point including noise. *Past* it, among the many interpolating solutions, SGD finds minimum-norm-like ones that get smoother, and test error descends again.
- Operational rules:
  - If a model interpolates train data and test error is bad, the classical move (shrink the model) and the modern move (grow it, or add data/regularization) can *both* work — you may be at the peak. Try more data or more params before less.
  - **Epoch-wise double descent exists**: test error can rise then fall again with more training. Do not early-stop solely on the first validation-loss uptick for large models on noisy labels; look at a longer horizon.
  - Label noise amplifies the peak. Clean your labels before concluding "the model is too big."
  - Never invoke "more params than data points → will overfit" as an argument. It's empirically false for modern nets; say instead "check the train-test gap and where you sit relative to interpolation."

## Regularizers: what each one actually does

| Method | Actual mechanism | When it's the right tool | Trap |
|---|---|---|---|
| Weight decay (AdamW) | shrinks weights toward 0 *decoupled* from grad adaptivity; limits effective capacity and interacts with LR (effective strength ∝ lr·wd) | default-on for transformers (~0.01–0.1) | L2-in-the-loss ≠ AdamW's decoupled decay under Adam — they behave differently; `Adam(weight_decay=)` in old semantics was L2, use AdamW. Don't decay layernorm gains/biases. |
| Dropout | trains an implicit ensemble of subnetworks; at test time weights scaled to match expectation (inverted dropout scales at train) | small-data finetuning, dense heads | forgetting `model.eval()` at inference (dropout still active → noisy, worse outputs) is a top-3 real-world bug; also dropout before BatchNorm shifts BN statistics — order matters |
| Data augmentation | expands the training distribution with label-preserving transforms; injects invariance priors directly | almost always the highest-ROI regularizer when applicable | augmentations must preserve the label for *your* task (see computer-vision skill); augmenting val/test data invalidates evaluation |
| Early stopping | limits how far optimization travels from init ≈ constraint on effective complexity | cheap, safe default with a val set | epoch-wise double descent can make the "best" early-stop point misleading on noisy labels; keep the curve, not just the argmin |
| Label smoothing | soft targets cap logit magnitudes, prevents overconfidence | classification with clean-ish labels | breaks calibration-sensitive downstream use of logits and hurts knowledge distillation from the smoothed model |

Rule: these are not interchangeable knobs. Augmentation adds information; the others only restrict capacity. Reach for augmentation/data first, weight decay second, dropout last for large models (modern LLMs train with dropout 0).

## Failure modes & pitfalls

- **Shape errors presented confidently**: dL/dW = gᵀx (transposed result), missing `.sum(0)` on bias grads, Hadamard-vs-matmul confusion for activations. Always re-derive with explicit shapes.
- **Passing probabilities to `F.cross_entropy`** (it expects raw logits) or stacking `softmax` + `CrossEntropyLoss` → double softmax: trains, converges slowly, silently caps confidence. Symptom: loss plateaus near a suspiciously moderate value.
- **Diagnosing NaNs as "exploding gradients" generically.** Order of checks: (1) loss function numerics (log(0), division by zero in custom losses), (2) LR/warmup, (3) fp16 overflow (use bf16 or loss scaling), (4) bad data batch (grad-norm spike on a specific step is data; steady growth is LR/architecture).
- **"BatchNorm fixes internal covariate shift"** as an explanation — outdated; the defensible claim is landscape smoothing + scale control. Also: BatchNorm with batch size < ~8 has garbage statistics (use GroupNorm/LayerNorm); BatchNorm in an RNN/transformer over variable lengths leaks across positions — that's why transformers use LayerNorm.
- **Tuning LR without retuning weight decay** (their product is what matters under AdamW), or copying wd=0.1 to a tiny-data finetune where it fights the little signal you have.
- **Comparing optimizers/architectures at a single LR.** Any such comparison is invalid; sweep LR per variant (the best-LR-per-method comparison is the only fair one).
- **Freezing BatchNorm incorrectly during finetuning**: `requires_grad=False` stops the affine params but running stats still update in train mode. You must also call `.eval()` on BN modules (or `track_running_stats` handling). Symptom: good finetune metrics, degraded performance on the original domain, non-reproducible eval.
- **Trusting a smooth loss curve while the model is broken**: dead ReLUs (fraction of zero activations per layer > ~50%), collapsed embeddings, or one layer's grad norm at machine epsilon coexist with decreasing loss. Log activation/grad stats.
- **Gradient accumulation ≠ larger batch under BatchNorm or with per-batch normalization losses** (e.g., contrastive losses computed within-batch): statistics/negatives are computed per microbatch. Equivalence holds only for purely per-sample losses.
- **Averaging loss wrongly under accumulation/DDP with variable-length batches**: `reduction='mean'` averages per token/sample within each microbatch, so microbatches with fewer valid tokens get overweighted after summing. For masked losses, sum losses and divide by the *global* valid-token count.
- **Chasing metric ghosts across "identical" runs** without seeding dataloader workers (`worker_init_fn`, `generator=`) and CUDA — augmentation randomness alone produces run-to-run gaps larger than many claimed improvements.
- **Misreading "train loss < val loss" as overfitting evidence when regularization differs between modes**: dropout and augmentation are active only in training, so train loss is measured on a harder problem; it can legitimately sit *above* val loss, and a small gap says nothing by itself. Compare val metrics over time, not the raw gap.
- **Interpreting a lower final loss from a larger batch run as "better"** without noting the LR schedule and total tokens/samples seen were held fixed — batch changes step count for the same epochs; compare at equal data seen and tuned LR.

## Worked micro-example: numerical gradient check (the tool, not the idea)

```python
import torch

def gradcheck_scalar(f, w, i, eps=1e-6):
    """Check df/dw[i] for scalar loss f(w). Use float64 or eps noise dominates."""
    w = w.double().requires_grad_(True)
    loss = f(w); loss.backward()
    analytic = w.grad.flatten()[i].item()
    with torch.no_grad():
        wp = w.detach().clone(); wp.flatten()[i] += eps
        wm = w.detach().clone(); wm.flatten()[i] -= eps
        numeric = (f(wp) - f(wm)).item() / (2 * eps)
    rel_err = abs(analytic - numeric) / max(abs(analytic), abs(numeric), 1e-12)
    return analytic, numeric, rel_err   # rel_err < 1e-6 in float64 = pass
```
Expert usage notes: check in float64 (float32 gives rel_err ~1e-3 even for correct gradients — do not "fix" a correct implementation to chase that); check a handful of random indices, not all; check at a *generic* point (ReLU kinks and max ops make finite differences wrong exactly at non-differentiable points — perturb inputs slightly if rel_err fails only sporadically); for a full module use `torch.autograd.gradcheck(fn, inputs, eps=1e-6, atol=1e-4)` with double-precision inputs.

## Worked micro-example: predicting a vanishing-gradient failure

Config: 20-layer MLP, tanh, Xavier init, no residuals/norm, lr 1e-3. Predict before running: tanh'(z) ≤ 1 with typical value ~0.65 for unit-scale inputs; Xavier keeps weight-Jacobian singular values ~1, so per-layer gradient scale ~0.65 → at layer 1: 0.65²⁰ ≈ 1.8e-4. First-layer LR is effectively 1e-3 × 1.8e-4 — early layers barely move; the net trains like a 3-layer MLP on frozen random features. Fix options in order of expected effect: add residual connections (restores ~flat grad norms), switch to ReLU+He (φ' ∈ {0,1}), add LayerNorm. Verify the diagnosis by logging `p.grad.norm()` per layer: expect ~3 orders of magnitude decay pattern before the fix, flat after.

## Verification / self-check

1. Any hand-derived gradient: check shapes term-by-term, then check one scalar numerically — `(L(w+ε) − L(w−ε))/2ε` vs autograd, tolerance ~1e-5 in float64.
2. Any training-dynamics claim: state the observable that would confirm it (per-layer grad norms, update/weight ratio, activation saturation fraction) — if you can't name the observable, the claim is folklore.
3. Any "it will overfit/generalize" prediction: identify the regime (under- vs over-parameterized, label noise level) explicitly first.
4. Any optimizer/regularizer comparison: confirm LR was swept per arm and schedules ended (not mid-decay) before endorsing the conclusion.
5. Numerics: confirm losses are computed on logits, mixed-precision uses bf16 or loss-scaling, and eval code calls `model.eval()` + `torch.no_grad()`.
