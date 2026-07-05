---
name: neural-network-training
description: Load when writing or debugging neural network training runs — loss not decreasing, loss spikes/NaNs, choosing learning rates and schedules, mixed-precision issues, gradient clipping, batch-size scaling, suspicious training curves, or "my model trains but performs worse than it should."
---

# Neural Network Training: Expert Corrections and Anchors

Assumed baseline (don't re-derive): staged sanity checks, initial loss = ln(C), one-batch overfit before real runs, sqrt-LR scaling for Adam batch changes (linear for SGD in vision), bf16 > fp16 where hardware allows, global-norm clipping, `scaler.unscale_` before clipping under AMP, checkpoints needing optimizer/scheduler/RNG state. This sheet is the sharpenings and misses.

## Startup protocol (run in order, one line each)

1. Fixed seed; log loss/LR/grad-norm/param-norm/throughput from step 0.
2. Eyeball 3 decoded post-transform batches; confirm input–label pairing survives shuffling (independent RNG states on two shuffles → permuted labels → model trains exactly to base rate and stops).
3. Initial loss = ln(C) (10 classes → 2.303; 50k-vocab LM → ≈10.8). Materially **lower** at step 0 = leakage — treat too-good loss with the same suspicion as divergence.
4. Overfit one batch to ~0 loss, regularization off. First check when stuck: did the optimizer receive all params? `sum(p.numel() for g in opt.param_groups for p in g["params"])` vs model total — a forgotten `model.head` silently never trains.
5. Brief full-data run: train loss must fall below the one-batch-memorization ceiling; val tracks train initially. Only then scale.

## Corrections the strong-baseline reflex gets wrong

- **Clipping launders non-finite gradients into silent no-ops.** With an `inf` grad-norm, `clip_grad_norm_` computes clip-scale 0 and zeroes that step's update — the run continues and the bad batch is never found. The reflex "clipping handles spikes" hides exactly the steps you must investigate. Check `torch.isfinite(grad_norm)` explicitly; skip and log those steps.
- **Clipping active on most steps is not "working."** It rescales your effective LR randomly per step and masks a real problem (LR too high, bad data, unstable loss). Log the *pre-clip* norm; healthy runs clip on a small minority of steps. Fix the cause; clipping is a shock absorber, not a suspension.
- **`F.cross_entropy` accepts wrong targets silently.** One-hot float targets `(N, C)` are interpreted as class *probabilities* — trains, just wrong. MSE on integer class indices also trains. Neither raises. Assert targets are int64 `(N,)` once per pipeline.
- **Label smoothing moves the loss floor.** With smoothing on, minimum loss > 0 — "won't reach zero" is not a bug, and the one-batch overfit check must run with smoothing off or it fails spuriously.
- **A plateaued one-batch overfit at a specific value is a diagnosis, not noise**: stuck at ln(C) with grad_norm ≈ 0 → detached/frozen params; stuck at ln(C) with healthy grads → LR/scheduler or label bug; plateau at an intermediate value (say 0.7) → duplicate inputs with conflicting labels, or pads leaking into the loss mean — count non-ignored targets per batch (`ignore_index=-100`) to confirm.
- **Gradient accumulation costs k× wall-clock per optimizer step.** It's a memory trick, not free throughput; if the larger effective batch isn't measurably helping, stop paying for it. Clip once on the accumulated gradient, never per micro-batch.
- **A spike that recovers vs. one that doesn't need different responses.** Recovers: rare bad batch or LR×curvature transient — replay the logged batch index if frequent. Permanently higher: Adam's second moment absorbed the spike (state poisoned) — restart from the pre-spike checkpoint; the run rarely regains its old trajectory on its own.

## Loss-curve pathology → first checks (ordered by prior)

| Symptom | Check first |
|---|---|
| Flat at init value from step 0 | `param_groups[0]["lr"]` (scheduler stepped per-batch instead of per-epoch or warmup pinning LR≈0); per-layer grad-norms (detach/frozen/missing params) |
| Plateaus far above expected | LR 10× too low; degenerate/duplicated targets; smoothing/masking accounting — capacity is the *last* suspect |
| Much slower than reference | LR low; missing input normalization (raw 0–255 pixels trains, badly, silently) |
| NaN in first steps | LR; missing Adam warmup; fp16 overflow; log(0)/0-div in custom loss |
| NaN deep into run | fp16 reduction overflow; degenerate batch (empty seq → NaN mean); replay checkpoint + exact batch |
| Train falls, val flat from eval #1 | Not overfitting — broken val pipeline / distribution mismatch / shuffled val labels. Overfitting *turns* upward after improving |
| Val below train persistently large | val leaked into train (small gaps: dropout/augmentation active in train only — benign) |
| Epoch-aligned sawtooth | no reshuffle: `shuffle=True` and, with DDP, `DistributedSampler.set_epoch()` — the latter is missed even when shuffle is "on" |

## LR and schedule anchors (exact values)

- Range test: pick **1/3–1/10 of the explosion LR** (steepest-descent region). The curve's minimum is already at the edge of instability, and smoothed single-batch losses understate full-run variance — "the minimum of the range-test curve" is the classic miscalibration.
- Warmup: linear 0→peak over ~1–3% of total steps; mandatory for Adam-family at transformer scale.
- The **final** LR ≪ peak matters more than schedule shape (cosine-to-10%-of-peak is the robust default). A run that "plateaued" often just needed its decay phase; never compare checkpoints mid-decay against fully-decayed ones.
- AdamW: effective regularization = lr × weight_decay — retuning LR silently retunes decay. Don't decay norms/biases.
- Init deltas: init classifier/regression heads small (std ~0.02) or zero the last layer of each residual branch; a std-1.0 head → giant initial logits → wrong initial loss and possible immediate divergence.

## Mixed precision (fp16-specific sharpenings)

- fp16 overflow lives in **reductions inside custom ops** (variance over a long sequence, softmax sum, large dot products) — autocast protects library ops; upcast custom reductions with `.float()` manually.
- NaN bisection: rerun the failing step in fp32 — persists = logic bug, vanishes = precision bug. Do this before reading any code.
- Continuously collapsing GradScaler scale = genuine instability elsewhere, not a scaler problem; occasional early "skipping step" messages are normal.
- Pure-fp16 weights (no fp32 master copy) stall late in training: updates below ~1e-3 relative round away.

## Reproducibility, checkpoints, throughput (one-liners)

- Checkpoint = weights + optimizer + scheduler + step counters + RNG states (torch/CUDA/NumPy/Python) + GradScaler; write atomically (temp file + rename) — a job preempted mid-`torch.save` corrupts the only checkpoint.
- Seeds ≠ GPU determinism: need `torch.use_deterministic_algorithms(True)`, `cudnn.deterministic=True`, `benchmark=False`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`; changing num_workers/batch/GPU-count changes RNG consumption. Any claim under ~1% needs ≥3 seeds or determinism mode.
- Expected tokens/sec = peak FLOPs × MFU(0.3–0.5) / 6N — compute before long runs; at 5% utilization fix the pipeline first.
- nvidia-smi 100% ≠ efficient; the classic serializer is a per-step `.item()`/`.cpu()`/print forcing a host sync — log every N steps or accumulate on-device.
- Adopt `torch.compile`/fused optimizers/flash-attention *after* the pipeline is verified at baseline; a miscompiled path is one more suspect while debugging.

## Verification checklist

- [ ] Initial loss = ln(C); one-batch overfit ≈ 0 (smoothing off).
- [ ] Decoded batches inspected; pad masking verified by counting non-ignored targets.
- [ ] Pre-clip grad-norm logged; non-finite grads detected explicitly, not clipped away.
- [ ] LR logged over time and matches intended schedule; decay phase completes before judging.
- [ ] Custom-op reductions upcast if fp16; resume tested against an unbroken curve.
- [ ] Improvements backed by ≥3 seeds or determinism mode.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 1 delta (expanded).
- Biggest baseline gaps found: clip_grad_norm_ silently zeroing updates on inf grad-norm (baseline treats clipping as the handler for spikes); F.cross_entropy's silent acceptance of one-hot float targets as probabilities; label-smoothing loss floor invalidating the overfit check.
- Baseline was otherwise strong (startup protocol, LR range test, determinism flags, 6N throughput, fp16 overflow sites all produced cold) — this skill is now a correction sheet, not a survey.
