---
name: neural-network-training
description: Load when writing or debugging neural network training runs — loss not decreasing, loss spikes/NaNs, choosing learning rates and schedules, mixed-precision issues, gradient clipping, batch-size scaling, suspicious training curves, or "my model trains but performs worse than it should."
---

# Neural Network Training: Running and Debugging

## Core mental model

- **Training bugs are silent.** Unlike ordinary software, a broken training pipeline usually still trains — loss goes down, nothing crashes, and the model is just 20% worse than it should be. Wrong labels, doubled augmentation, off-by-one target shifts, and BatchNorm-in-eval-mode all produce *plausible* curves. Therefore: verify by construction (staged sanity checks) rather than by inspection.
- **Overfit a single batch first, always.** Before any real run: take 1 batch (or ~10 samples), disable augmentation/dropout/weight decay, train until loss ≈ 0 (classification: literally memorize; LM: loss near the entropy floor). If the model cannot memorize 10 samples, there is a bug — in the loss, targets, masking, or architecture wiring — and no full run will fix it. This single check catches the majority of pipeline bugs in minutes.
- **The most common "model bug" is a data loader bug.** When a model underperforms mysteriously, suspect the data pipeline before the architecture, the optimizer, or the hyperparameters. Look at actual decoded batches with your eyes.
- **LR is the hyperparameter; nearly everything else is second-order.** A 3× LR error costs more than any architecture tweak gains. Budget tuning effort accordingly: get LR right, then epochs/schedule, then maybe weight decay — the rest is usually noise at typical scales.
- **Know your expected initial loss.** Cross-entropy over C balanced classes must start at ≈ ln(C) (10 classes → 2.303; 50k-token vocab LM → ≈10.8). Materially higher: broken init or wrong input scaling. Materially lower at step 0: leakage or a degenerate task. This one number, checked at step 0, validates loss wiring, init, and label encoding simultaneously.

## The startup protocol (run in this order, skip nothing)

1. Fixed seed; log everything (loss per step, LR, grad-norm, param-norm, throughput) from step 0.
2. **Inspect decoded data**: visualize/print 3 batches *after* all transforms — images post-normalization, token IDs decoded back to text, labels alongside inputs. Confirm input–label pairing survives shuffling (a classic: inputs and labels shuffled with different RNG states → permuted labels; model trains to base rate and no further).
3. Check initial loss = theoretical value (above).
4. **Overfit one batch to ~zero loss.** If stuck: check loss masking/reduction, target dtype/shape (see pitfalls), and that the optimizer actually received all parameters (`sum(p.numel() for g in opt.param_groups for p in g["params"])` vs model total — a forgotten `model.head` silently never trains).
5. Train briefly on full data; verify train loss falls below the one-batch-memorization ceiling and val tracks train initially.
6. Only now scale up and tune.

## Loss-curve pathology → diagnosis

| Symptom | Likely causes, in order of prior probability |
|---|---|
| Flat at initialization value from step 0 | LR ≈ 0 (scheduler bug: `scheduler.step()` per epoch called per batch, or warmup misconfigured so LR stays ~0); frozen/detached params (a stray `.detach()`, `requires_grad=False`, or params missing from optimizer); gradients zeroed after compute. Check `optimizer.param_groups[0]["lr"]` and per-layer grad-norms *first* — 30 seconds, finds most of these. |
| Decreases then plateaus far above expected | LR too low (try 10×); data pipeline feeding degenerate/duplicated targets; model capacity truly saturated (rare — believe this last); loss saturated by label smoothing/masking accounting rather than real progress. |
| Slow steady decrease, much slower than references | LR too low; batch too small for noisy gradients; input normalization missing (raw 0–255 pixels into a conv net trains, just badly — a classic silent bug). |
| Sudden spike, then recovery | A rare bad batch (corrupt sample, extreme-length outlier), or LR × curvature transient. Occasional small spikes are normal, especially early. Frequent spikes: lower LR, or clip gradients, and hunt the specific batch (log batch indices; replay the spiking step). |
| Spike then *permanently higher* loss | The optimizer state got poisoned (Adam second-moment blown up by a huge gradient) or weights left a good basin: lower LR, add/lower clipping, and restart from the pre-spike checkpoint — the run rarely recovers to its old trajectory by itself. |
| Divergence to NaN/inf within first steps | LR far too high; missing warmup with Adam (Adam's early variance estimates are unreliable — use warmup ≥ a few hundred steps); exploding logits from bad init; fp16 overflow (see mixed precision); division by zero in a custom loss (log(0), 0/0 in a normalization). |
| NaN appearing mid-run, deep into training | fp16 overflow (loss scale climbed, then a big activation); a rare degenerate batch (empty sequence → 0-length mean = NaN); numerically unsafe op hit at the tail of a distribution (log/ sqrt/ softmax without max-subtraction in custom code). Bisect by replaying the checkpoint + the exact batch. |
| Train falls, val flat from the start (not diverging later — flat *always*) | This is not overfitting — it's leakage-free memorization of train with zero generalization: usually train/val from different distributions, broken val pipeline (wrong normalization at eval), or val labels shuffled. Overfitting looks like val improving then *turning* upward. |
| Val loss below train loss | Usually benign: dropout/augmentation active in train only, or train loss averaged over the epoch while val measured at epoch end. Large persistent gaps: val set leaked into train, or val is easier. |
| Periodic sawtooth aligned with epochs | Data not reshuffled each epoch (`shuffle=True` on the DataLoader / `set_epoch()` on DistributedSampler forgotten) — model tracks the fixed ordering. |

## Learning rate: the knob that matters

- **LR range test** to pick the initial LR: sweep LR exponentially from ~1e-7 upward over a few hundred steps, plot loss vs LR; loss falls, bottoms, explodes. Choose ~1/3 to 1/10 of the explosion LR (the steepest-descent region, not the minimum of the curve — the minimum is already unstable).
- **Warmup**: linear 0→peak over ~1–3% of total steps (a few hundred to a few thousand). Mandatory with Adam-family at transformer scale; skipping it is the most common cause of early-run divergence.
- **Schedules**: cosine-to-~10%-of-peak is the robust default; linear decay is fine; step decay is legacy. What matters far more than the shape: the *final* LR being much lower than peak (final ≈ peak leaves loss noisy and high — a run that "plateaued" often just needed its decay phase).
- **Batch size ↔ LR scaling**: when you multiply batch size by k, scale LR up too — linear scaling (×k) works for SGD in vision up to a few thousand; for Adam, sqrt(k) is the safer prior; in all cases re-warmup and re-verify stability. Corollary: changing batch size for memory reasons and keeping LR fixed *is* a hyperparameter change — expect different results, don't blame the model.
- Adam epsilon and betas rarely need touching; weight decay interacts with LR (AdamW decouples them — use AdamW, and know that in AdamW the actual decay applied is `lr × weight_decay`, so changing LR changes effective regularization too).

## Initialization and normalization interplay

- Modern defaults (Kaiming/Xavier init + BatchNorm/LayerNorm) mean you rarely hand-tune init — *except*: custom layers added without thought (an output head initialized with std 1.0 → giant initial logits → huge initial loss and possible immediate divergence; init final classifier/regression heads small, e.g. std 0.02, or zero the final layer of each residual branch so the network starts near identity).
- Residual networks: per-block output scaling or zero-init of the last norm/layer in each block stabilizes deep stacks; if you build a deep custom net that diverges instantly at conservative LR, this is the first structural fix.
- **BatchNorm foot-guns**: (1) `model.eval()` forgotten at validation → val metrics computed with batch statistics — noisy and wrong (and the reverse, eval() left on during training, silently freezes running stats); (2) batch size < ~8 makes BN statistics garbage — switch to GroupNorm/LayerNorm; (3) BN + DistributedDataParallel computes per-GPU statistics — fine at large per-GPU batch, harmful at small (use SyncBatchNorm).
- LayerNorm placement in transformers: pre-LN trains stably at depth without careful warmup; post-LN (original) reaches slightly better final loss but diverges easily — if reproducing an old post-LN recipe, its long warmup is load-bearing, don't trim it.

## Gradient clipping — and when it hides bugs

- Clip by **global norm** (`torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)`), not per-parameter value. Typical max-norm 0.5–1.0 for transformers.
- Log the *pre-clip* grad-norm every step. Healthy runs clip occasionally (spikes, early steps). **If clipping activates on most steps, clipping is not "working" — it is rescaling your effective LR randomly per step and hiding a real problem** (LR too high, bad data, numerically unstable loss). Fix the cause; clipping is a shock absorber, not a suspension.
- Clipping masks NaN-adjacent bugs poorly: `inf` grad-norm makes the clip scale 0, silently zeroing that step's update. Check for non-finite grad-norm explicitly and skip/investigate those steps rather than letting clip launder them.
- With gradient accumulation: clip once on the *accumulated* gradient before `optimizer.step()`, not per micro-batch. With AMP: `scaler.unscale_(optimizer)` before clipping, else you're clipping scaled gradients against an unscaled threshold (a no-op or near-total clip depending on the scale).

## Mixed precision pitfalls

- **Prefer bf16 wherever hardware allows** (Ampere+): same exponent range as fp32 → no loss scaling, no overflow drama. Most fp16 folklore exists because fp16's max is 65,504.
- fp16 requires **dynamic loss scaling** (`torch.cuda.amp.GradScaler`): small gradients underflow to zero in fp16; the scaler multiplies loss up, then unscales grads. Symptoms of scaler trouble: many "skipping step, reducing scale" messages early is normal; *continuously* collapsing scale means genuine instability elsewhere.
- **Overflow lives in reductions**: variance in a norm over a long sequence, sum in softmax, a large dot product — each can exceed 65,504 in fp16 even when all inputs are small. Keep norms, softmax, and loss computation in fp32 (autocast does this for library ops; *custom* kernels/ops are where fp16 overflow bugs actually happen — upcast reductions manually with `x.float()`).
- Master weights in fp32 (AMP does this via the optimizer): pure-fp16 weights stall late in training because updates smaller than ~1e-3 relative get rounded away.
- Don't validate numerics in half precision: when debugging a NaN, first rerun the failing step in full fp32; if the NaN persists it's a logic bug, if it disappears it's a precision bug — this bisection saves hours.

## Data loader bugs (the most common "model" bug)

- Input–label desync via independent shuffles or a non-deterministic transform applied before pairing.
- Augmentation applied to val/test, or normalization *not* applied at eval (train normalized, eval raw → mysterious eval gap).
- `num_workers > 0` with a naive per-worker RNG: every worker applies identical "random" augmentations, or NumPy seeds duplicate across workers so epochs repeat the same crops (seed per worker via `worker_init_fn` using `torch.initial_seed()`).
- Off-by-one target shift in LM training (labels must be inputs shifted by one — doing it twice, or not at all, both "train" with plausible-looking loss; not-at-all converges absurdly fast to near-zero loss because the answer is in the input — suspiciously *good* loss is a bug signal too).
- Padding leaking into loss: mean over all tokens including pads dilutes the loss and gradient — mask pads (`ignore_index=-100` in `F.cross_entropy`) and verify the *count* of non-ignored targets per batch is what you expect.
- Class targets with wrong dtype/shape: `F.cross_entropy` wants class *indices* `(N,)` int64 — one-hot floats of shape `(N, C)` are interpreted as class probabilities and train, just wrong; and MSE on class indices trains too. Both are silent.

## Regularization and generalization knobs (after the pipeline is correct)

- Order of resort when val lags train: more data / better augmentation > early stopping on val metric > weight decay (AdamW 0.01–0.1) > dropout > architectural shrinking. Reaching for dropout while the LR schedule is wrong is treating symptoms.
- Augmentation must respect label semantics: horizontal flips break digit/text/chirality tasks; aggressive crops can cut out the labeled object entirely (audit augmented samples against labels, same eyeball ritual as data loading).
- Label smoothing (0.1) is a cheap, usually-safe win for classification — but it changes the loss floor: with smoothing ε over C classes the minimum achievable loss is no longer 0, so don't chase "loss won't reach zero" as a bug after enabling it, and the one-batch overfit check must be run with smoothing off.
- Early stopping needs patience measured in *validations*, not epochs, and the val metric that matters — val loss and val accuracy frequently disagree late in training (loss rises from growing confidence on errors while accuracy still improves); stop on the deployment metric.

## Checkpointing and reproducibility

- A resumable checkpoint = model weights + **optimizer state** + LR-scheduler state + step/epoch counters + RNG states (torch, CUDA, NumPy, Python) + the AMP scaler state. Resuming Adam without its moment estimates causes a loss bump and a subtly different trajectory; resuming without scheduler state restarts warmup at the current step.
- **Seeds do not guarantee determinism on GPU.** cuDNN autotuning picks different algorithms run-to-run (`torch.backends.cudnn.benchmark=True`), several CUDA kernels use non-deterministic atomics (scatter/gather, some backward passes), and reduction order varies with parallel decomposition. For true determinism: `torch.use_deterministic_algorithms(True)`, `cudnn.deterministic=True`, `benchmark=False`, env `CUBLAS_WORKSPACE_CONFIG=:4096:8` — at a real throughput cost, so use it for debugging, not production runs.
- Consequence: run-to-run variance is nonzero even with fixed seeds; never attribute a small metric difference between two runs to a code change without either determinism mode or multiple seeds (report mean ± std over ≥3 seeds for any claim under ~1%).
- Changing `num_workers`, batch size, or GPU count changes the data order and RNG consumption — bitwise reproduction requires those pinned too.
- Save checkpoints atomically (write temp file, then rename) — a preempted job that dies mid-`torch.save` corrupts the only checkpoint otherwise.

## Throughput and the "is it even training efficiently" check

- Compute a rough expected throughput before long runs: for transformers, model FLOPs per token ≈ 6 × N_params (fwd+bwd); divide hardware peak FLOPs × an achievable utilization (0.3–0.5 for well-tuned training) by that to get tokens/sec. If you're at 5% utilization, you have a pipeline problem worth fixing before burning GPU-weeks.
- GPU near 100% in `nvidia-smi` does not mean efficient — it counts any kernel activity. Profile one: if step time is dominated by data loading (GPU idle gaps between steps in the profiler timeline), raise `num_workers`, enable `pin_memory=True`, move CPU-heavy augmentation to GPU or pre-compute it.
- The classic throughput bug: an accidental CPU–GPU sync every step (`loss.item()`, `.cpu()`, or printing a tensor inside the step) serializing the pipeline. Log scalars every N steps, not every step, or accumulate on-device.
- Gradient accumulation is a memory trick, not free: k accumulation steps ≈ k× wall-clock per optimizer step. If a bigger effective batch isn't demonstrably helping (measure!), don't pay for it.
- `torch.compile` / fused optimizers / flash attention are large real wins on transformer workloads — but adopt them *after* the pipeline is verified correct at baseline; a miscompiled or fused path is one more suspect during debugging.

## Worked micro-example: the one-batch overfit harness

```python
model.train()
batch = next(iter(train_loader))
x, y = batch["input"].cuda(), batch["label"].cuda()
opt = torch.optim.AdamW(model.parameters(), lr=3e-4)

import math
print("expected init loss:", math.log(NUM_CLASSES))   # e.g. 2.303 for 10 classes
for step in range(500):
    loss = F.cross_entropy(model(x), y)
    opt.zero_grad(); loss.backward()
    gn = torch.nn.utils.clip_grad_norm_(model.parameters(), 1e9)  # measure, don't clip
    opt.step()
    if step % 50 == 0: print(step, loss.item(), "grad_norm", gn.item())
```

Interpretation: loss should hit < 0.01 within a few hundred steps. Stuck at ~2.3 with grad_norm ≈ 0 → gradients not flowing (detach/frozen params). Stuck at ~2.3 with healthy grad_norm → LR/scheduler or label bug. Plateaus at, say, 0.7 → part of the batch is unlearnable (duplicate inputs with conflicting labels — check for exactly that) or loss masking is including padding. Passes cleanly → the core pipeline is sound; scale up.

## Worked micro-example: LR range test

```python
import copy, math
probe = copy.deepcopy(model)          # never run the sweep on your real weights
opt = torch.optim.AdamW(probe.parameters(), lr=1e-7)
lrs, losses, gamma = [], [], (1e-1 / 1e-7) ** (1 / 300)   # 1e-7 -> 1e-1 over 300 steps
it = iter(train_loader)
for step in range(300):
    x, y = next(it)
    loss = F.cross_entropy(probe(x.cuda()), y.cuda())
    opt.zero_grad(); loss.backward(); opt.step()
    lrs.append(opt.param_groups[0]["lr"]); losses.append(loss.item())
    for g in opt.param_groups: g["lr"] *= gamma
    if not math.isfinite(losses[-1]) or losses[-1] > 4 * min(losses): break
```

Reading the plot (loss vs log-LR): flat region at tiny LR (too small to move), a descending slope, a minimum, then explosion. Suppose loss starts descending at 3e-5, bottoms near 1e-3, explodes past 3e-3. Pick peak LR ≈ 2e-4–5e-4 (steep-descent region, 1/3–1/10 of explosion) — not 1e-3, because the minimum of this curve sits at the edge of instability and the smoothed single-batch losses understate variance over a full run. Add warmup to that peak and cosine decay to ~1/10 of it. Total cost: ~2 minutes of GPU time to replace days of guess-and-check.

## Verification checklist before blaming the model or reporting results

- [ ] Initial loss equals the theoretical value; one-batch overfit reaches ~zero.
- [ ] Decoded post-transform batches inspected by eye; label pairing verified; pad masking verified by counting non-ignored targets.
- [ ] LR sanity: range test done or known-good recipe copied *including* warmup and decay; `param_groups[0]["lr"]` logged and matches intent over time.
- [ ] Grad-norm (pre-clip) logged; clipping active on a small minority of steps only; non-finite grads detected explicitly.
- [ ] `model.eval()` / `model.train()` correct at each phase; eval preprocessing identical to train (minus augmentation).
- [ ] Mixed precision: bf16 if available; if fp16, scaler state monitored and custom reductions upcast.
- [ ] Checkpoints include optimizer/scheduler/RNG/scaler state; resume tested by comparing a resumed curve against an unbroken one.
- [ ] Any claimed improvement backed by multiple seeds or determinism mode, not one lucky run.
