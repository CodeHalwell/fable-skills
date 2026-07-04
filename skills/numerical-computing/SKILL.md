---
name: numerical-computing
description: Loads when writing or debugging floating-point code — choosing fp64/fp32/fp16/bf16, diagnosing NaN/Inf/precision loss, comparing floats with tolerances, stabilizing formulas (variance, log-sum-exp, softmax, quadratic roots), summation/accumulation error, log-space probability math, or explaining nondeterminism and failed equality tests in numerical results.
---

# Floating-Point and Numerical Stability

## Core mental model

1. **Floats are a fixed budget of *relative* precision.** A binary float is sign × significand × 2^exp: fp64 has 52+1 significand bits (~15.9 decimal digits, machine ε ≈ 2.2e-16), fp32 has 23+1 (~7.2 digits, ε ≈ 1.2e-7), fp16 has 10+1 (~3.3 digits, ε ≈ 9.8e-4, max ≈ 65504), bf16 has 7+1 (~2.4 digits, ε ≈ 7.8e-3, but fp32's exponent range). Every arithmetic op rounds: `fl(a op b) = (a op b)(1 + δ)`, |δ| ≤ ε. All stability analysis is bookkeeping of these δ's.
2. **Cancellation is the killer, not rounding.** Subtracting nearly equal numbers is exact — but it *promotes* previously negligible relative errors into the leading digits. `(1e8 + 1) − 1e8` in fp32 gives 0, not 1. The cure is never "more precision" first; it's algebraic reformulation so the subtraction of near-equals never happens.
3. **Conditioning is the problem's fault; stability is the algorithm's fault.** An ill-conditioned problem (κ large) amplifies input error no matter the algorithm. An unstable algorithm ruins even a well-conditioned problem. Diagnose which you have before "fixing": rule of thumb error ≈ κ × ε_machine for a backward-stable algorithm. If κ·ε already explains your error, no algorithm change will help — reformulate the problem or raise precision.
4. **Probabilities live in log space.** Products of thousands of probabilities underflow fp64 around e^−745. Store log p, add instead of multiply, and re-enter linear space only through log-sum-exp. This isn't an optimization — it's the only correct implementation for likelihoods, HMMs, softmax losses.
5. **Floating-point addition is not associative**, so any change in summation order — threading, GPU atomics, different BLAS, different batch split — legally changes the result. Bitwise reproducibility across parallel hardware requires *forcing* an order (deterministic algorithms flags), and costs performance. Design tests and comparisons assuming last-digits noise exists.

## Decision frameworks

### Precision selection
| Context | Choice | Reasoning |
|---|---|---|
| Scientific computing, linear algebra, optimizers' internal state | fp64 | κ up to ~10⁸ still leaves 8 digits |
| NN training weights/activations | bf16 (mixed with fp32 master weights/accumulators) | Same exponent range as fp32 → no overflow drama; the lost mantissa is tolerable for gradients |
| NN training on hardware without bf16 | fp16 + loss scaling | fp16's max 65504 makes overflow easy and gradients (~1e-7) underflow; loss scaling shifts them into range |
| Accumulating many terms (sums, dot products, batch stats) | Accumulate in fp32/fp64 even when data is fp16/bf16 | Accumulation error grows with n; low-precision accumulators are the classic mixed-precision bug |
| Money | Never binary floats — integers of cents or `decimal.Decimal` | 0.1 is not representable; pennies leak |
| Comparing/testing | Same dtype as computation, tolerances scaled to it | rtol must be ≥ dtype ε by a healthy factor |

### Tolerance picking (the actual numbers)
- Use `np.allclose(a, b, rtol, atol)` semantics: passes if |a−b| ≤ atol + rtol·|b|. **rtol** handles the relative-precision budget: start at `1e-7` for fp64 pipelines (∼1000·ε slack for a few dozen ops), `1e-4`–`1e-3` for fp32, `1e-2` for fp16/bf16. Multiply by ~√n or n for results accumulated over n ops if failing legitimately.
- **atol** exists only for values near zero, where rtol vanishes; set it to (expected magnitude of your data) × rtol, *not* the default. The NumPy default `atol=1e-8` silently passes garbage when your data's scale is 1e-6 (everything is "close to" everything), and fails correct results when the scale is 1e+12.
- Asymmetry trap: `allclose(a, b) != allclose(b, a)` because rtol scales |b| only. For symmetric checks use `math.isclose` (symmetric) or compare |a−b| ≤ rtol·max(|a|,|b|) yourself.
- Never `assert a == b` for floats except: exact small integers, values copied not computed, or deliberate bitwise-reproducibility tests.

### Cancellation: recognize → reformulate (standard rewrites)
| Dangerous form | When it bites | Stable rewrite |
|---|---|---|
| `(-b + sqrt(b²-4ac)) / 2a` | b² ≫ 4ac; the "+" root cancels | Compute `q = -(b + sign(b)·sqrt(b²-4ac))/2`; roots are `q/a` and `c/q` |
| `E[x²] − E[x]²` for variance | mean ≫ std (e.g. data ~10⁶ ± 1: fp64 keeps ~4 digits; fp32 returns garbage/negative) | Two-pass: `mean((x−x̄)²)`; streaming: Welford's algorithm |
| `log(1+x)`, `exp(x)−1` for small x | x ≲ 1e-8: `1+x` rounds to 1 | `np.log1p(x)`, `np.expm1(x)` |
| `1 − cos(x)` for small x | cos(x) ≈ 1 | `2·sin(x/2)²` |
| `sqrt(x+1) − sqrt(x)` for large x | near-equal subtraction | `1/(sqrt(x+1)+sqrt(x))` |
| `log(sum(exp(z)))` | overflow at z>709 (fp64), underflow z<−745 | `m + log(sum(exp(z−m)))`, m = max(z) — this is `scipy.special.logsumexp` |
| `a·b/c` with huge/tiny factors | intermediate over/underflow though result is fine | Reorder `(a/c)·b`, or go through logs |

### Softmax / cross-entropy specifics
- Softmax must subtract the row max before exp: `exp(z − z.max())`. Without it, any logit > ~88 overflows fp32 to inf → inf/inf → NaN. Subtracting the max is mathematically a no-op (softmax is shift-invariant) and bounds every exp argument by 0.
- Never compute `log(softmax(z))` as two steps — small probabilities round to 0 and log gives −inf. Use the fused log-softmax: `z − logsumexp(z)`. Same reason cross-entropy losses take *logits*, not probabilities (`torch.nn.CrossEntropyLoss`, `F.binary_cross_entropy_with_logits`): the fused forms cancel the exp/log analytically.

## Failure modes & pitfalls

- **"Fix precision problems by switching to fp64."** Buys ~9 digits once; a cancellation that loses digits proportionally still loses them. Reformulate first; raise precision only when κ·ε genuinely explains the error and the problem can't be restated.
- **Testing `A @ inv(A) == I` or comparing matrices with `==`.** Fails legitimately at ~κ·ε per entry. Compare `np.linalg.norm(A @ Ainv - np.eye(n)) / np.linalg.norm(A)` against κ(A)·ε × (modest factor). Any "matrix equality" check must be a normwise relative tolerance.
- **Summing a million fp32 values with a naive loop or fp16 accumulator.** Naive summation error grows like O(n·ε)·Σ|xᵢ|; at n=10⁶, fp32 can lose ~half its digits, and adding 1.0 to a fp32 running total that has reached 2²⁴ = 16,777,216 does *nothing* (a real bug class in fp16/fp32 training-step counters and metric accumulators). Fixes in preference order: `np.sum` (pairwise, error ~O(log n)); `math.fsum` (exact); Kahan summation when writing custom loops/kernels:
  ```python
  s = c = 0.0
  for x in xs:
      y = x - c          # compensated add
      t = s + y
      c = (t - s) - y    # recovers the rounding error just made
      s = t
  ```
  Beware: compilers with `-ffast-math` may "optimize" Kahan's correction away — it algebraically cancels.
- **Underflow silently at zero, then log/divide.** `np.exp(-800)` is 0.0 with no warning; a later `log` gives −inf, a division gives inf, and the NaN surfaces three functions later. Trace NaNs *backwards to the first inf/underflow*, not where they appear. Use `np.errstate(all='raise')` or `torch.autograd.set_detect_anomaly(True)` to catch at origin.
- **`x - x.mean()` not being exactly zero-mean.** The residual mean is ~ε·|x̄| — fine, unless you then multiply by 1e15 or assert `== 0`. Centering, orthogonalization, and normalization are approximate; re-normalize after long chains (e.g., re-orthogonalize Q every k Gram–Schmidt steps; classical Gram–Schmidt is unstable, use modified GS or Householder QR).
- **Equality-comparing across devices/threads and calling it a bug.** GPU reductions, atomics, and different BLAS builds sum in different orders; last-2-3-digit differences between CPU/GPU or run/run are *expected*. For reproducibility in PyTorch: `torch.use_deterministic_algorithms(True)`, set `CUBLAS_WORKSPACE_CONFIG=:4096:8`, seed everything — and accept the slowdown. Do not chase bitwise equality across different hardware; define correctness by tolerance.
- **Mixed dtype creep.** A single fp64 constant in a JAX/NumPy expression can upcast (or in JAX's default fp32 world, silently *not*), and one fp16 tensor downcasts a sum. Check `result.dtype` at boundaries; in PyTorch AMP, know which ops autocast to fp16 (matmuls) vs stay fp32 (reductions, norms) — the design is exactly the accumulate-in-high-precision rule.
- **Interpreting `0.1 + 0.2 != 0.3` class bugs as library errors** — or worse, "fixing" with `round(x, 2)` scattered everywhere. Decimal-looking behavior needs decimal types; numerical code needs tolerances. Rounding for display only.
- **Catastrophic loss in finite differences.** Gradient check with h too small: `(f(x+h) − f(x))/h` has cancellation error ~ε/h vs truncation error ~h. Optimum h ≈ √ε·scale for forward differences (~1e-8 in fp64), ε^(1/3) (~6e-6) for central. h = 1e-15 gives pure noise — a perennial "my analytic gradient is wrong" false alarm.
- **ULP reasoning absence.** Adjacent fp64 numbers near 1.0 differ by ~2.2e-16, near 1e16 by ~2.0 — *there are no fp64 values with fractional parts beyond 2⁵³*; integer IDs above 2⁵³ (common with JSON → float parsing of snowflake IDs!) silently corrupt. Check any pipeline that routes big integers through floats.

## Worked micro-examples

**1. Variance, three ways, with real numbers.** Data: 10⁶ samples ~ N(10⁶, 1) in fp32.
- Naive `mean(x²) − mean(x)²`: x² ≈ 10¹², fp32 has ~7 digits → representation error per term ~10⁵, utterly swamping the true variance of 1. Output: huge, random, possibly negative.
- Two-pass `mean((x − mean(x))²)`: after centering, values are O(1); result ≈ 1.0 to fp32 precision. Correct.
- Welford (streaming, one pass): maintains running mean and M2 via `delta = x − mean; mean += delta/n; M2 += delta*(x − mean)`; matches two-pass stability without a second pass. Use for streaming/batch-norm-style statistics.

**2. Log-sum-exp mechanics.** z = [1000, 1001, 999] (fp64). Naive: `exp(1000)` = inf → nan. Shifted: m = 1001, `exp([−1, 0, −2]) = [0.3679, 1.0, 0.1353]`, sum = 1.5032, answer = 1001 + log(1.5032) = 1001.4076. Softmax falls out as `exp(z − 1001.4076)` = [0.2447, 0.6652, 0.0900]. Every probability, every likelihood sum, every mixture-model E-step goes through this exact move.

**3. Quadratic formula rescue.** x² − 1e8·x + 1 = 0 (fp64). True roots ≈ 1e8 and 1e-8. Naive small root: `(1e8 − sqrt(1e16 − 4))/2`; `sqrt(1e16−4)` rounds to exactly 1e8 → small root computed as 0.0 (100% error). Stable: q = −(−1e8 + sign(−1e8)·1e8)/... → big root 1e8 via q/a, small root via c/q = 1/1e8 = 1e-8, both full precision. The product-of-roots identity (Vieta) replaced the cancelling subtraction — the template for all cancellation fixes.

## Verification & self-check

- **Estimate the error budget before trusting output**: (number of accumulated ops) × ε × (condition number of the step). If observed error ≫ budget, there's a bug; if ≈ budget, the computation is as good as it gets.
- **Perturbation probe for conditioning**: rerun with inputs jittered by relative 1e-12 (fp64). Output changes of 1e-4 mean κ ≈ 10⁸ — report that sensitivity instead of pretending 15 digits.
- **Precision A/B**: run the same computation in fp32 and fp64; digits that agree are trustworthy. Cheap, catches both cancellation and accumulation issues.
- **Scan for inf/underflow at the source**: `np.isfinite(x).all()` checkpoints after each stage; first failing stage owns the bug.
- **Test with adversarial scales**: exercise code at data scales 1e-8, 1, 1e8 and mean≫std regimes; stable formulas pass all three, naive ones fail loudly.
- **Check the reference, not just self-consistency**: compare against `math.fsum`/`mpmath` (arbitrary precision) on a small instance to know the *true* answer when validating a summation or special-function implementation.
