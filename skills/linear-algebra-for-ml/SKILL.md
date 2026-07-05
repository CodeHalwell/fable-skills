---
name: linear-algebra-for-ml
description: Loads when working with matrix computations in ML contexts — debugging shape/broadcasting errors, choosing decompositions (SVD/eig/QR/Cholesky), reasoning about conditioning, curvature, or learning rates, writing einsum, implementing PCA/least-squares/attention, or deciding whether to invert, solve, or factor.
---

# Linear Algebra as Practiced in ML

Most of this domain is strong-model baseline; this sheet keeps the routing checklist plus the anchors and corrections that get fumbled cold.

## Routing checklist (one-liners; completeness is the point)

- SPD solve → Cholesky (`cho_factor`/`cho_solve`); repeated RHS → factor once (~n³/3), then 2n² per RHS — batch RHS into one matrix for BLAS-3.
- General square → `np.linalg.solve` (LU). Never `inv(A) @ b`; explicit inverse only when you need many entries of the inverse itself (e.g., estimator covariance).
- Least squares → QR/SVD `lstsq`; **never form AᵀA** — κ(AᵀA) = κ(A)². Same trap disguised: eigendecomposing `X.T @ X / n` for PCA instead of SVD of centered X.
- Symmetric eig → `eigh` (symmetrize `(A+A.T)/2` first), never `eig`. Top-k of huge/sparse → `eigsh`/`svds`/`randomized_svd`.
- Log-likelihood determinant → `slogdet` or 2·Σ log diag(L); `det` under/overflows past ~100 dims (`det(0.1·I₃₀₀)` = 0 for a perfectly conditioned matrix).
- Tall SVD → pass `full_matrices=False`; the NumPy default allocates an m×m U (10⁶×100 input → 8 TB).
- `pinv`, `matrix_rank`, `cond` each run a full SVD — hoist out of loops.
- Sampling N(μ,Σ) → one Cholesky, x = μ + Lz; don't call `multivariate_normal` per draw in a loop.

## Anchors worth having exactly

- **GD/eigenvalue arithmetic:** stable iff η < 2/λmax; per-step contraction c ⇒ ~1/(1−c) steps per e-fold, ~ln10/(1−c) per decimal digit. c = 0.98 (κ=100 at optimal step) → ~50 steps/e-fold, ~115/digit. (Unit slip alert: earlier revisions of this file quoted the per-digit figure as "per e-fold".) The LR is pinned by λmax, progress by λmin — that gap *is* κ; second-order/adaptive methods attack the ratio, not the scale.
- **Memory before math:** fp64 n×n = 8n² bytes; n = 50,000 → 20 GB. Cost any "kernel/attention matrix" plan first.
- **κ budget:** a backward-stable solve loses ≈ log₁₀κ digits. If κ·ε already explains observed error, changing solvers won't help — reformulate (center, rescale, orthogonalize; ridge caps effective κ at ~σ₁²/λ). Most horrific κ values are a unit mismatch between columns and vanish after standardization.
- **Sparse crossover:** below ~95% zeros, sparse formats usually *lose* to dense BLAS (indexing overhead > flop savings); measure before converting. CSR for rows/matvec, CSC for columns, COO/LIL to build; never grow CSR incrementally (O(nnz) per insert). If you only ever apply the operator, use `LinearOperator(shape, matvec=f)` into `eigsh`/`svds`/`cg`. CG iterations scale with √κ — precondition when κ ≫ 10⁴.
- **Structure before generic routines:** `d[:, None] * A`, never `np.diag(d) @ A`. Woodbury solves (A+UVᵀ)⁻¹ in O(n²k) with an existing factor — but repeated low-rank updates accumulate error; use a proper Cholesky rank-1 update or refactor every few hundred updates. (A⊗B)vec(X) = vec(BXAᵀ) — never build a Kronecker product.
- **PSD-by-math, indefinite-by-rounding** (eigenvalues like −3e-9 from an fp32 covariance): clip at 0 or add jitter `1e-9·tr(A)/n·I`; know which downstream needs which — sampling needs strict PD, so jitter. Don't `abs()`.

## Corrections for the recurring silent bugs

- `(n,) − (n,1)` → (n,n), not elementwise: the loss computes, the model trains, the numbers are nonsense. Assert shapes at function boundaries; write the shape signature `(B,T,d) @ (d,k) -> (B,T,k)` as a comment before the expression.
- `x / x.sum(axis=1)` breaks for 2-D x — use `keepdims=True`. Insert axes with `[:, None]` explicitly rather than relying on lucky alignment.
- `np.dot` on 3-D arrays contracts the last axis of A with B's *second-to-last*, producing an outer-batched monster; `@` does sane batched matmul. Anything batched: write the einsum.
- einsum: an index absent from the output is **summed**; always write the explicit `->`. Pass `optimize=True` for chains — contraction *order* changes complexity class (`A@(B@v)` is O(n²) where `(A@B)@v` is O(n³)).
- `np.linalg.matrix_rank` default tol scales with σ₁: columns spanning 1e-6..1e6 get genuine directions zeroed. Scale columns or pass explicit `tol`.
- Eigenvectors are defined up to sign and up to rotation within repeated-eigenvalue subspaces: compare `min(‖v1−v2‖, ‖v1+v2‖)` or subspace principal angles, never raw vectors across runs/libraries.
- Projection onto span of non-orthonormal W is `W @ solve(W.T@W, W.T@y)` (or QR) — `W @ (W.T @ y)` is valid only when WᵀWᵀ = I; symptom: the "removed" direction still correlates with the residual.
- Dual-vs-primal PCA (`X@X.T` (n,n) vs `X.T@X` (p,p)): picking the smaller Gram is legitimate, but converting eigenvectors requires v = Xᵀu/σ — skipping the conversion yields subtly wrong components. And PCA always needs centering first.
- Huge-norm solution with small residual = near-singular A (differenced or duplicate columns). Regularize or drop the redundant column; switching solvers won't fix the problem's κ.
- Ridge in spectral form: filter factor s²/(s²+λ) per singular direction — one SVD gives the whole regularization path (`w = Vt.T @ ((s/(s**2+lam)) * (U.T@y))` for every λ).

## Verification & self-check

- Reconstruction check after any decomposition: `‖A − U diag(s) Vt‖/‖A‖ < 1e-10` (fp64) catches transposes/ordering/economy-vs-full in one line.
- Report the relative residual `‖Ax−b‖/(‖A‖‖x‖+‖b‖)`, not "it ran"; small residual + huge ‖x‖ = ill-conditioning, say so.
- `‖QᵀQ − I‖` for anything claimed orthonormal; re-orthogonalize if it drifts past ~1e-10 in long iterations.
- After any reshape/transpose chain, run a tiny tensor whose entries encode their own coordinates (`np.arange(24).reshape(2,3,4)`) and verify element (b,i,j) lands where claimed — the batch/feature interleave bug is invisible otherwise.
- Invariance tests localize bugs: PCA invariant to row permutation; least squares invariant to duplicating a row at half weight; violations point at centering/weighting/broadcasting.
- Print singular values on a log scale before any rank/conditioning claim: clean gap → trustworthy rank; smooth decay → "rank" is a modeling choice, present it as such.
- Gradient-check hand-derived matrix calculus against central differences (rtol ≤ 1e-5, fp64); the two standard analytic errors are a dropped transpose and a factor of 2 from symmetric terms.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 12 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus 4.8 reproduces this domain cold — factorization choice, κ² normal-equations trap, full_matrices, Sherman–Morrison with costs, broadcasting rules, eigh-vs-eig, clip-vs-jitter — often with refinements beyond the old text (batched POTRS, cholupdate).
- The audit's main find was internal: the old worked example quoted "~120 steps per e-fold" for figures that are per decimal digit (per e-fold ≈ 50). Retained value is the routing checklist + the recurring-silent-bug corrections, not exposition.
