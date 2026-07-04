---
name: numerical-computing
description: Loads when writing or debugging floating-point code — choosing fp64/fp32/fp16/bf16, diagnosing NaN/Inf/precision loss, comparing floats with tolerances, stabilizing formulas (variance, log-sum-exp, softmax, quadratic roots), summation/accumulation error, log-space probability math, or explaining nondeterminism and failed equality tests in numerical results.
---

# Floating-Point and Numerical Stability

## Core mental model

1. **Floats are a fixed budget of *relative* precision.** A binary float is sign × significand × 2^exp. fp64: 52+1 significand bits (~15.9 decimal digits, machine ε ≈ 2.2e-16, range to ~1.8e308). fp32: 23+1 bits (~7.2 digits, ε ≈ 1.2e-7, max ~3.4e38). fp16: 10+1 bits (~3.3 digits, ε ≈ 9.8e-4, max 65504 — overflow is a daily hazard). bf16: 7+1 bits (~2.4 digits, ε ≈ 7.8e-3) but fp32's exponent range — trades digits for never overflowing where fp32 wouldn't. Every op rounds: `fl(a op b) = (a op b)(1 + δ)`, |δ| ≤ ε. Stability analysis is bookkeeping of these δ's.
2. **Cancellation is the killer, not rounding.** Subtracting nearly equal numbers is itself exact — but it *promotes* previously negligible relative errors into the leading digits, and deletes the information that distinguished the operands. `(1e8 + 1) − 1e8` in fp32 returns 0, not 1. The cure is never "more precision" first; it's algebraic reformulation so the near-equal subtraction never happens.
3. **Conditioning is the problem's fault; stability is the algorithm's fault.** An ill-conditioned problem amplifies input error no matter the algorithm; an unstable algorithm ruins even well-conditioned problems. Rule of thumb: error ≈ κ × ε_machine for a backward-stable algorithm. If κ·ε already explains your observed error, no algorithm change helps — reformulate the problem or raise precision. If observed error ≫ κ·ε, the algorithm (or code) is unstable — fix it, don't throw bits at it.
4. **Probabilities live in log space.** Products of thousands of probabilities underflow fp64 near e^−745. Store log p, add instead of multiply, and return to linear space only through log-sum-exp. This is not an optimization; it's the only correct implementation of likelihoods, HMM/CRF recursions, and softmax losses.
5. **Floating-point addition is not associative**, so any change in summation order — threading, GPU atomics, a different BLAS, a different batch split — legally changes the result. Bitwise reproducibility on parallel hardware requires forcing an order (deterministic-algorithms flags) and costs speed. Design every test and comparison assuming last-digit noise exists.
6. **Integers hide inside floats — up to a cliff.** fp64 represents all integers exactly up to 2⁵³ ≈ 9.0e15; beyond that, consecutive integers are not representable. fp32's cliff is 2²⁴ = 16,777,216. Any pipeline that routes big IDs or counters through floats (JSON parsers, DataFrame type coercion) silently corrupts values past the cliff.

## Decision frameworks

### Precision selection
| Context | Choice | Reasoning |
|---|---|---|
| Scientific computing, linear algebra, optimizer internal state | fp64 | κ up to ~10⁸ still leaves ~8 digits |
| NN training weights/activations | bf16, mixed with fp32 master weights and fp32 accumulation | fp32 exponent range → no overflow drama; lost mantissa is tolerable for gradients |
| NN training on fp16-only hardware | fp16 + loss scaling | fp16 max 65504 overflows easily; small gradients underflow — loss scaling shifts them into range |
| Accumulating many terms (sums, dots, batch stats) | fp32/fp64 accumulator even when data is fp16/bf16 | Accumulation error grows with n; low-precision accumulators are *the* classic mixed-precision bug |
| Money | Never binary floats — integer cents or `decimal.Decimal` | 0.1 has no finite binary representation; pennies leak |
| Timestamps / large IDs | int64, never float | 2⁵³ cliff (see above) |
| Comparing/testing | Same dtype as computation, tolerances scaled to that dtype's ε | rtol must exceed dtype ε by a healthy factor |

### Tolerance picking (actual numbers)
- `np.allclose(a, b, rtol, atol)` passes when |a−b| ≤ atol + rtol·|b|. **rtol** carries the relative-precision budget: start at `1e-7` for fp64 pipelines (~1000ε of slack for a few dozen ops), `1e-4`–`1e-3` for fp32, `1e-2` for fp16/bf16. Loosen by ~√n (random error) up to n (systematic) for results accumulated over n ops.
- **atol** exists only for values near zero, where the rtol term vanishes. Set it to (expected data magnitude) × rtol — *never* leave NumPy's default `atol=1e-8` unexamined: with data at scale 1e-6 everything passes (atol dwarfs the values); with data at scale 1e12 correct results fail.
- Asymmetry trap: `allclose(a, b) ≠ allclose(b, a)` since rtol scales |b| only. For symmetric checks use `math.isclose` (symmetric by design) or write |a−b| ≤ rtol·max(|a|,|b|).
- For arrays, prefer a normwise check `‖a−b‖/‖b‖ ≤ rtol` when the object is a vector/matrix as a whole; elementwise checks over-fail on entries that are individually tiny but irrelevant.
- Never `assert a == b` on floats except: small exact integers, values copied rather than computed, or deliberate bitwise-reproducibility tests.

### Cancellation: recognize → reformulate (standard rewrites)
| Dangerous form | When it bites | Stable rewrite |
|---|---|---|
| `(-b + sqrt(b²-4ac)) / 2a` | b² ≫ 4ac: the "+" root cancels | `q = -(b + sign(b)·sqrt(b²-4ac))/2`; roots are `q/a` and `c/q` (Vieta) |
| `E[x²] − E[x]²` for variance | mean ≫ std | Two-pass `mean((x−x̄)²)`; streaming: Welford |
| `log(1+x)`, `exp(x)−1`, small x | x ≲ 1e-8: `1+x` rounds to 1 | `np.log1p(x)`, `np.expm1(x)` |
| `1 − cos(x)`, small x | cos ≈ 1 | `2·sin(x/2)²` |
| `sqrt(x+1) − sqrt(x)`, large x | near-equal subtraction | `1/(sqrt(x+1)+sqrt(x))` — multiply by the conjugate |
| `log(sum(exp(z)))` | overflow z > 709 (fp64) / underflow z < −745 | `m + log(sum(exp(z−m)))`, m = max(z) — `scipy.special.logsumexp` |
| `sqrt(x² + y²)` | x or y near the overflow boundary | `np.hypot(x, y)` — scales internally |
| `a·b/c`, huge/tiny factors | intermediate over/underflow though the result is fine | reorder `(a/c)·b`, or route through logs |
| `mean = sum(x)/n` then `sum(x - mean)` as “zero” check | residual is ~n·ε·|x̄|, not 0 | compare against that budget, not against 0.0 |

### Softmax / cross-entropy specifics
- Softmax must subtract the row max before exp: `np.exp(z - z.max(axis=-1, keepdims=True))`. Any logit > ~88 overflows fp32 → inf/inf → NaN. Subtracting the max is mathematically free (softmax is shift-invariant) and bounds every exponent argument by 0.
- Never compute `log(softmax(z))` in two steps — small probabilities round to 0.0 and log yields −inf. Fused log-softmax is `z − logsumexp(z)`. This is why loss APIs take *logits* (`torch.nn.CrossEntropyLoss`, `F.binary_cross_entropy_with_logits`): the fused forms cancel exp/log analytically and are stable at extreme confidence.
- Sigmoid at large |x|: `1/(1+exp(-x))` overflows for x ≪ 0; use the two-branch form or `scipy.special.expit`. log-sigmoid is `-np.logaddexp(0, -x)` — never `log(sigmoid(x))`.

### Summation method selection
| Scale | Method | Error growth |
|---|---|---|
| < ~10³ same-sign terms, fp64 | naive loop is fine | O(n·ε) but n is small |
| Any n, NumPy available | `np.sum` (pairwise) | ~O(log n · ε) |
| Exactness required (tests, money-adjacent, ill-conditioned series) | `math.fsum` | exactly rounded |
| Custom kernel/accumulator, low-precision data | Kahan (or Neumaier variant, robust when terms exceed the running sum) | O(ε), independent of n |
| Mixed signs with cancellation | sort by magnitude ascending, or fsum | ordering matters most when the true sum is small vs Σ|xᵢ| |
| GPU/parallel reductions | tree reduction (library default) in fp32+ accumulator | deterministic only with deterministic flags |

### Determinism playbook (when "same code, different answer" is filed)
1. Same machine, same run, different results → real race or uninitialized memory: an actual bug, chase it.
2. Same machine, run-to-run differences in last digits → nondeterministic reduction order (GPU atomics, thread scheduling), dropout/seed, or hash-order-dependent iteration. Decide whether you need bitwise determinism; if yes: seed everything, `torch.use_deterministic_algorithms(True)`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`, fixed dataloader order/workers, and accept ~10–30% slowdown; some ops simply have no deterministic implementation and must be avoided.
3. CPU vs GPU, or different GPU models, or different BLAS/MKL versions → expected tolerance-level divergence; assert with rtol scaled to dtype, never bitwise.
4. Same binary, different results across OS/compilers at the last ULP → x87/FMA/vectorization differences; treat like case 3.
Never "fix" case 3/4 divergence by loosening tolerances until everything passes — compute the legitimate budget (ops × ε × κ) and investigate anything beyond it.

### NaN/Inf autopsy workflow (run it in this order)
1. **Identify the species.** Inf comes from overflow (`exp`, products, division by tiny) — it still obeys ordering. NaN comes from undefined ops: inf − inf, 0·inf, inf/inf, 0/0, sqrt/log of negative, and *any* op touching an existing NaN. A NaN at step k usually means an Inf or a domain violation at step k−j.
2. **Bisect to the first non-finite tensor/array**, not the first NaN loss: checkpoint `np.isfinite(x).all()` (or forward hooks in torch) between stages; the first failing stage owns the bug.
3. **Classify the origin**: exp/softmax overflow (missing max-subtraction) · log/sqrt of a value that underflowed to 0 or went −1e-9 negative (missing clamp/jitter) · division by a variance/norm that collapsed to 0 (missing ε in the denominator — note ε goes *inside* the sqrt for rsqrt-style normalizers: `x/sqrt(v+ε)`, not `x/(sqrt(v)+ε)` — both appear in the wild and behave differently at v≈0) · fp16 range overflow (loss scaling / bf16) · bad input data (NaNs in the raw features; always check first).
4. **Fix at the origin, not the symptom.** `torch.nan_to_num` or `np.nan_to_num` at the surface point hides the bug and silently corrupts gradients/statistics; it is a last-resort output sanitizer, never a fix.
5. Remember NaN's comparison semantics while debugging: `NaN != NaN` is true, `sorted()` with NaNs is undefined-order garbage, `np.nanmax` exists for a reason, and `x == x` is the cheap NaN test.

### Condition numbers of elementary steps (where digits actually die)
- Subtraction of near-equals: κ = |x|/|x−y| → unbounded; the *only* elementary op that's arbitrarily ill-conditioned. All the classic rewrites exist to dodge it.
- `exp(x)`: relative error in output ≈ |x| × (relative error in x) — at x = 700, six input digits become zero output digits. Large-argument exponentials are information amplifiers; keep computations in log space until the last step.
- `log(x)` near 1: κ = 1/|log x| → large; this is precisely why `log1p` exists (compute log(1+δ) from δ directly).
- Polynomial roots, matrix eigenvalues with defective/clustered spectra: tiny coefficient changes move answers a lot — report sensitivity, don't chase digits.
- sin/cos at huge arguments: argument reduction mod 2π needs the *absolute* precision your float no longer has at 1e10 — `sin(1e10)` in fp64 carries only ~6 meaningful digits. Rephrase the phase to stay small.

## Failure modes & pitfalls

- **"Fix precision problems by switching to fp64."** Buys ~9 extra digits exactly once; a cancellation that loses digits proportionally still loses them. Reformulate first; raise precision only when κ·ε genuinely explains the error and the problem cannot be restated.
- **Testing `A @ inv(A) == I` or matrix equality with `==`.** Legitimately fails at ~κ·ε per entry. Compare `norm(A @ Ainv - I)/norm(A)` against κ(A)·ε times a modest factor. Every "matrix equality" must be a normwise relative tolerance — this is also why cached vs recomputed results, or CPU vs GPU results, differ without either being wrong.
- **Summing a million fp32 values naively, or accumulating in fp16.** Naive loop summation error grows ~O(n·ε)·Σ|xᵢ|. Worse cliff: adding 1.0 to an fp32 running total that has reached 2²⁴ = 16,777,216 does *nothing* — real bug class in step counters and metric accumulators. Fix order of preference: `np.sum` (pairwise summation, error ~O(log n)); `math.fsum` (exactly rounded); Kahan compensated summation for custom loops/kernels:
  ```python
  s = c = 0.0
  for x in xs:
      y = x - c          # compensated add
      t = s + y
      c = (t - s) - y    # recover the rounding error just committed
      s = t
  ```
  Caveat: `-ffast-math`-style compiler flags may delete Kahan's correction — it is algebraically zero.
- **Underflow silently to zero, then log/divide.** `np.exp(-800)` → 0.0 with no warning; a later `log` → −inf; a division → inf; the NaN surfaces three functions downstream. Trace NaNs *backwards to the first inf or underflow*, not to where they appear. Instrument with `np.errstate(all='raise')`, `np.seterr`, or `torch.autograd.set_detect_anomaly(True)`.
- **Gram–Schmidt drift.** Classical Gram–Schmidt loses orthogonality (‖QᵀQ−I‖ grows with κ); modified GS is better, Householder QR is right. In long iterative processes (Lanczos, orthogonal RNN tricks), re-orthogonalize periodically instead of assuming orthogonality persists.
- **Equality-comparing across devices/threads/runs and filing a bug.** GPU reductions, atomics, and different BLAS builds sum in different orders; last-digit differences between CPU/GPU or run/run are *expected*. For true reproducibility in PyTorch: `torch.use_deterministic_algorithms(True)`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`, seed everything, single-threaded data order — and accept the slowdown. Across *different* hardware, define correctness by tolerance, never bitwise.
- **Mixed-dtype creep.** One fp64 constant upcasts a NumPy expression; one fp16 tensor downcasts a sum; JAX defaults to fp32 unless `jax_enable_x64` is set (a classic "results differ from NumPy" report). Check `result.dtype` at function boundaries. In torch AMP, know the split: matmuls autocast to fp16/bf16; reductions, norms, and softmax stay fp32 — the design *is* the accumulate-in-higher-precision rule.
- **`0.1 + 0.2 != 0.3` class bugs "fixed" with `round(x, 2)` sprinkled everywhere.** Decimal-looking behavior needs decimal types (`decimal.Decimal`, integer cents); numerical code needs tolerances. Round for display only.
- **Finite differences with h too small.** `(f(x+h) − f(x))/h` has cancellation error ~ε/h against truncation error ~h. Optimum h ≈ √ε·scale ≈ 1e-8 for fp64 forward differences; ε^{1/3} ≈ 6e-6 for central. h = 1e-15 yields pure noise — the perennial false alarm "my analytic gradient must be wrong".
- **ULP blindness at scale.** Spacing between adjacent fp64 values is ~2.2e-16 near 1.0 but ~2.0 near 1e16: an absolute tolerance of 1e-9 is 10⁷ ULPs near 1.0 (way too loose) and 0 ULPs near 1e16 (impossible to satisfy). This is exactly why tolerances must be relative, and why `atol` is only for the neighborhood of zero.
- **Catastrophic parentheses.** `x*x - y*y` loses to cancellation when x≈y; `(x-y)*(x+y)` is stable. `(a+b)+c ≠ a+(b+c)` matters when a ≈ −b: add the small terms first. Sorting by increasing magnitude before summing reduces error for same-sign series.
- **Denormal/subnormal slowdowns.** Values below ~1.2e-38 (fp32) enter the subnormal range; on many CPUs each op on them is 10–100× slower — a "performance bug" that is actually numerical (fading signals, decaying filters). Flush to zero deliberately if the tail doesn't matter.

## Worked micro-examples

**1. Variance, three ways, with real numbers.** Data: 10⁶ samples ~ N(10⁶, 1) in fp32.
- Naive `mean(x²) − mean(x)²`: x² ≈ 10¹², and fp32's ~7 digits mean each x² carries representation error ~10⁵ — swamping the true variance of 1. Output: enormous, random, often negative (then `sqrt` → NaN, the downstream symptom).
- Two-pass `mean((x − mean(x))²)`: after centering, values are O(1); result ≈ 1.0 to fp32 precision.
- Welford streaming: `delta = x − mean; mean += delta/n; M2 += delta*(x − mean)`; matches two-pass stability in one pass — the correct pattern for streaming/batch statistics.
The naive formula is fine *only* when mean ≲ std; the check costs one comparison.

**2. Log-sum-exp mechanics.** z = [1000, 1001, 999] in fp64. Naive: `exp(1000)` = inf → nan. Shifted: m = 1001, `exp([−1, 0, −2]) = [0.3679, 1.0, 0.1353]`, sum = 1.5032, answer = 1001 + log(1.5032) = 1001.4076. Softmax falls out as `exp(z − 1001.4076) = [0.2447, 0.6652, 0.0900]`. Every likelihood sum, mixture-model E-step, and attention row goes through this exact move; `np.logaddexp` / `logaddexp2` are the two-argument forms for pairwise accumulation.

**3. Quadratic formula rescue.** x² − 1e8·x + 1 = 0 (fp64). True roots ≈ 1e8 and 1e-8. Naive small root: `(1e8 − sqrt(1e16 − 4))/2`, but `sqrt(1e16 − 4)` rounds to exactly 1e8 → small root computed as 0.0, a 100% relative error. Stable: q = −(b + sign(b)√(b²−4ac))/2 = 1e8-ish with no cancellation; roots q/a = 1e8 and c/q ≈ 1e-8, both to full precision. Vieta's product identity replaced the cancelling subtraction — the template for every cancellation fix: find an algebraic identity that computes the small quantity *directly* instead of as a difference of large ones.

**4. The 2²⁴ accumulator cliff, demonstrably.**
```python
import numpy as np
s = np.float32(16_777_216.0)
print(s + np.float32(1.0) - s)   # 0.0 — the +1 vanished
```
Any fp32 counter, running loss, or metric sum crossing 16.7M silently stops increasing by small increments. Same bug at 2⁵³ in fp64, and at 2048 for fp16 (where even +1 fails above 2¹¹). Accumulate in a wider type or in integers.

## Verification & self-check

- **Estimate the error budget first**: (ops accumulated) × ε × (condition number of the step). Observed error ≫ budget → bug; ≈ budget → the computation is as good as it can be, report the limit instead of "fixing".
- **Perturbation probe for conditioning**: rerun with inputs jittered by relative 1e-12 (fp64). Output moving by 1e-4 means κ ≈ 10⁸ — report the sensitivity rather than pretending 15 digits.
- **Precision A/B**: run the same computation in fp32 and fp64; the digits that agree are the trustworthy ones. Catches both cancellation and accumulation cheaply.
- **Scan for the first non-finite value**: `np.isfinite(x).all()` checkpoints between stages; the first failing stage owns the bug. Never debug a NaN where it surfaced.
- **Adversarial-scale tests**: exercise the code at data scales 1e-8, 1, 1e8, and in the mean ≫ std regime. Stable formulas pass all; naive ones fail loudly and early.
- **Ground truth from arbitrary precision**: validate summations and special functions against `math.fsum` / `mpmath` on small instances — self-consistency alone can't distinguish a stably-computed wrong answer.
- **State tolerances with their justification** ("rtol=1e-5: fp32 pipeline, ~100 accumulated ops") rather than copying defaults; a tolerance you can't justify is a tolerance that will either mask a bug or block a release at 2am.
