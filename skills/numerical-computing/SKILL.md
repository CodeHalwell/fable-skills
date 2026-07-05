---
name: numerical-computing
description: Loads when writing or debugging floating-point code — choosing fp64/fp32/fp16/bf16, diagnosing NaN/Inf/precision loss, comparing floats with tolerances, stabilizing formulas (variance, log-sum-exp, softmax, quadratic roots), summation/accumulation error, log-space probability math, or explaining nondeterminism and failed equality tests in numerical results.
---

# Floating-Point and Numerical Stability

Most of this domain is strong-model baseline (format layouts, Welford, Vieta, log-sum-exp, determinism flags, ε-inside-sqrt, fast-math-vs-Kahan). This sheet keeps the tables worth re-anchoring, the exact numbers, and the judgment rules.

## The three judgment rules (everything else follows)

1. **Conditioning is the problem's fault; instability is the algorithm's.** Error budget ≈ κ × ε_machine for a backward-stable algorithm. Observed error ≈ budget → the computation is as good as it can be; report the limit. Observed ≫ budget → the algorithm/code is unstable — fix it, don't throw bits at it. "Switch to fp64" is right only in the middle band (κ roughly 10⁷–10¹⁵ where fp32 is too small and fp64 suffices); it is a band-aid for cancellation, which reformulation removes outright.
2. **Cancellation, not rounding, is the killer** — subtraction of near-equals is the only elementary op with unbounded condition number. The fix is always an algebraic identity that computes the small quantity *directly* (Vieta for quadratics, Welford/two-pass for variance, log1p/expm1, conjugate for √(x+1)−√x, 2sin²(x/2) for 1−cos).
3. **Fix at the origin, not the symptom.** Trace NaNs backward to the first Inf/underflow/domain violation (`np.isfinite` checkpoints, `errstate(all='raise')`, `detect_anomaly`); `nan_to_num` at the surface hides the bug and corrupts statistics.

## Format table (know cold, quote exactly)

fp64: 52+1 bits, ε≈2.2e-16, ~15.9 digits, max ~1.8e308. fp32: 23+1, ε≈1.2e-7, ~7.2 digits, max 3.4e38. fp16: 10+1, ε≈9.8e-4, max **65504** — overflow is a daily hazard; needs loss scaling. bf16: 7+1, ε≈7.8e-3, fp32's exponent range — trades digits for never overflowing where fp32 wouldn't; the default on modern accelerators, with fp32 master weights and fp32 accumulation. Integer cliffs (consecutive integers stop being representable): 2⁵³ fp64, **2²⁴ = 16,777,216** fp32, **2048** fp16 — fp32 metric counters silently stop incrementing at 16.7M; IDs/timestamps go in int64, money in integer cents/Decimal.

## Tolerance discipline

- `allclose` passes on |a−b| ≤ atol + rtol·|b| — **asymmetric** (scales |b| only) and the default `atol=1e-8` is a landmine: data at scale 1e-6 all passes; use `math.isclose` or |a−b| ≤ rtol·max(|a|,|b|) for symmetric checks, set atol = (expected magnitude)×rtol, only for near-zero values.
- rtol starting points: 1e-7 fp64 pipelines, 1e-4–1e-3 fp32, 1e-2 fp16/bf16; loosen ~√n (random) to n (systematic) for n accumulated ops. Prefer normwise `‖a−b‖/‖b‖` for arrays-as-objects. State the justification ("rtol=1e-5: fp32, ~100 ops") — an unjustified tolerance masks a bug or blocks a release at 2am.
- Matrix "equality" legitimately fails at ~κ·ε per entry (`A @ inv(A)` vs I, CPU vs GPU, cached vs recomputed); compare normwise against κ·ε × modest factor. Never `==` on computed floats; quantize before using floats as dict keys/groups.

## Determinism triage (same code, different answer)

1. Same run, different results → real race/uninitialized memory: a bug.
2. Run-to-run last digits → reduction order (GPU atomics), seeds, hash order. If bitwise determinism is required: seed everything, `torch.use_deterministic_algorithms(True)`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`, fixed dataloader order; ~10–30% slower and some ops have no deterministic kernel.
3. CPU vs GPU / different BLAS → expected tolerance-level divergence; assert with dtype-scaled rtol.
4. Same binary, different OS/compiler, last ULP → FMA/vectorization; treat as 3. Never "fix" 3/4 by loosening tolerances until green — compute the ops×ε×κ budget and investigate only beyond it.

## Numbers that decide arguments

- Finite differences: forward h* ≈ √ε ≈ 1e-8 (fp64), central h* ≈ ε^⅓ ≈ 6e-6; h=1e-15 is pure noise — the perennial false "my analytic gradient is wrong" alarm.
- `exp` amplifies: relative output error ≈ |x|·(input error); at x=700, six input digits → zero output digits. Stay in log space until the last step. `sin(1e10)` fp64 has only ~6 meaningful digits (argument reduction eats absolute precision).
- Subnormals (<~1.2e-38 fp32): 10–100× slower per op on many CPUs — decaying-signal "performance bugs" are numerical; flush to zero deliberately.
- Summation: naive loop O(n·ε); `np.sum` pairwise ~O(log n·ε) (worst-case bound — not √n, which is the *random-walk* estimate for naive); `math.fsum` exactly rounded; Kahan/Neumaier O(ε) for custom loops — and `-ffast-math` deletes Kahan's correction (it's algebraically zero). Mixed signs: sort ascending by magnitude or fsum.
- Softmax/losses: subtract the row max (logits > ~88 overflow fp32); never two-step `log(softmax)` — fused `z − logsumexp(z)`; log-sigmoid = `-logaddexp(0, -x)`; this is *why* loss APIs take logits. ε goes **inside** the sqrt in normalizers (`x/sqrt(v+ε)`), else the gradient of √v blows up at v→0 — both forms appear in the wild.
- Underflow chain: `exp(-800)` → 0.0 silently → later log → −inf → division → NaN three functions downstream. Products of probabilities underflow fp64 near e⁻⁷⁴⁵ — likelihoods/HMM/CRF live in log space, non-negotiably.
- Drift: `t += dt` accumulates ε per step, `t = t0 + i*dt` doesn't; repeated incremental rotations leave the manifold — recompute from the original state or renormalize periodically.
- Clamp floors at the *computation's* underflow boundary (1e-12 before a log), not a comfortable round number (1e-3 destroys genuinely small probabilities and biases every downstream likelihood).
- JAX defaults to fp32 unless `jax_enable_x64` (classic "differs from NumPy" report); torch AMP keeps reductions/norms/softmax in fp32 by design — that *is* the accumulate-wider rule.

## Verification & self-check

- Error budget first (ops × ε × κ); precision A/B (fp32 vs fp64 — agreeing digits are the trustworthy ones); perturbation probe (jitter inputs by 1e-12, watch output move → measured κ).
- Adversarial-scale tests: run at 1e-8, 1, 1e8 and in the mean ≫ std regime; stable formulas pass all.
- Ground truth from `math.fsum`/`mpmath` on small instances — self-consistency can't distinguish a stably-computed wrong answer.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened: pairwise-summation error is O(log n·ε) worst case; the baseline offered the √n random-walk figure), 0 delta.
- Opus 4.8 nailed format layouts incl. bf16 rationale, Welford, Vieta, allclose asymmetry and the atol trap, the 2²⁴/2⁵³/2048 cliffs with round-to-even detail, NaN-origin workflow, h* for finite differences, log-sum-exp with worked numbers, exact PyTorch determinism flags, ε-inside-sqrt, sin(1e10)≈6 digits, fast-math-deletes-Kahan, subnormal slowdowns, logits-not-probabilities, and the κ-band rule for when fp64 helps.
- Retained value: the three judgment rules as a decision procedure, exact tolerance/budget discipline, determinism triage ladder, and checklist completeness.
