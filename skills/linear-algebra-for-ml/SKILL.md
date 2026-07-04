---
name: linear-algebra-for-ml
description: Loads when working with matrix computations in ML contexts — debugging shape/broadcasting errors, choosing decompositions (SVD/eig/QR/Cholesky), reasoning about conditioning, curvature, or learning rates, writing einsum, implementing PCA/least-squares/attention, or deciding whether to invert, solve, or factor.
---

# Linear Algebra as Practiced in ML

## Core mental model

1. **Shapes are the type system.** Every matrix bug is a shape bug you haven't found yet. Before writing any tensor expression, write the shape signature as a comment: `(B, T, d) @ (d, k) -> (B, T, k)`. Dimension bookkeeping catches ~80% of bugs before the first run — silent broadcasting catches the rest of you.
2. **SVD is the master decomposition.** `A = U Σ Vᵀ` gives you, for free: rank (count of nonzero σᵢ), best low-rank approximation (truncate — Eckart–Young says this is optimal in both Frobenius and spectral norm), PCA (SVD of centered data), pseudoinverse (`V Σ⁺ Uᵀ`), condition number (σ₁/σᵣ), null space and range (columns of V and U). When unsure which tool applies, ask "what does the SVD of this matrix look like?"
3. **Eigenvalues govern dynamics.** Anything iterated — power iteration, gradient descent, RNN state, Markov chains — is controlled by the spectrum of the iteration matrix. |λ|max < 1 means contraction/stability; a wide spread of eigenvalues means slow directions. Hessian eigenvalues are curvature: GD diverges when lr > 2/λmax, and converges at a rate set by κ = λmax/λmin.
4. **Never invert; solve or factor.** `inv(A) @ b` is slower, less accurate, and destroys structure (sparsity, symmetry). `solve(A, b)` uses a factorization directly. Explicit inverses are justified almost only when you need many entries of the inverse itself (e.g., the full covariance matrix of an estimator).
5. **Numerical rank ≠ mathematical rank.** In floating point, ask "how many singular values exceed tolerance?" not "is the determinant zero?" Determinants are useless as singularity tests: `det(0.1 * I_300)` underflows to 0 for a perfectly conditioned matrix, and `det` of an ill-conditioned matrix can be any size.
6. **Matrices are maps, not grids of numbers.** "What does this matrix do to a sphere?" (SVD: rotate, stretch by σᵢ, rotate) answers more design questions than any entrywise view. Rank = how many directions survive; norm = the biggest stretch; symmetric PSD = pure nonnegative stretching along orthogonal axes.

## Decision frameworks

### Which factorization?
| Situation | Use | Why |
|---|---|---|
| Solve Ax=b, A symmetric positive definite | Cholesky (`scipy.linalg.cho_factor`/`cho_solve`) | 2× faster than LU; fails loudly if not PD (a free PD test) |
| Solve Ax=b, general square A | LU via `np.linalg.solve` | Standard; backward stable with pivoting |
| Least squares, well-conditioned, m ≫ n | QR (`scipy.linalg.lstsq` with `lapack_driver='gelsy'`) | Never form AᵀA — that squares the condition number |
| Least squares, rank-deficient or ill-conditioned | SVD (`np.linalg.lstsq`, uses SVD with an `rcond` cutoff) or ridge | Pseudoinverse handles rank deficiency explicitly |
| Need top-k eigen/singular pairs of a huge/sparse matrix | Iterative: `scipy.sparse.linalg.eigsh`/`svds`, or `sklearn.utils.extmath.randomized_svd` | Full decomposition is O(n³); you only need k directions |
| Symmetric/Hermitian eigenproblem | `np.linalg.eigh`, never `eig` | Faster; real sorted eigenvalues; no spurious tiny imaginary parts |
| Repeated solves with same A, different b | Factor once (`lu_factor`, `cho_factor`), solve many | Factorization is O(n³); each subsequent solve is O(n²) |
| Determinant needed (e.g., Gaussian log-likelihood) | `np.linalg.slogdet`, or 2·Σ log(diag(L)) from Cholesky | `det` overflows/underflows beyond ~100 dims; log-det is what you actually need |
| Sampling from N(μ, Σ) | Cholesky: x = μ + L z | One factorization; `multivariate_normal` re-derives it per call if misused in a loop |

### Broadcasting rules, precisely (so you can predict instead of test)
- Alignment is **right-to-left**: compare trailing dimensions; each pair must be equal or one of them 1 (which stretches). Missing leading dims are treated as 1. `(8, 1, 6) op (7, 6) -> (8, 7, 6)`; `(3,) op (3, 1) -> (3, 3)` — the second one is the classic accident.
- `keepdims=True` on reductions exists to keep broadcasting aligned: `x / x.sum(axis=1)` breaks for 2-D x (shapes `(n,m)` vs `(n,)` align on the wrong axis); `x / x.sum(axis=1, keepdims=True)` is correct.
- Insert axes explicitly with `None`/`np.newaxis` rather than relying on implicit alignment: `a[:, None] * b[None, :]` announces the outer product; `a * b` with lucky shapes hides it.
- Broadcasting never copies memory (it's a stride trick), so it's free until an op materializes the result. `np.broadcast_to` + a reduction can often replace materializing a big intermediate; better yet, that pattern is usually an einsum.
- Batched matmul broadcasting: `(B, n, k) @ (k, m) -> (B, n, m)` works; so does `(B, 1, n, k) @ (1, H, k, m) -> (B, H, n, m)`. Leading dims broadcast; the last two are the matrix.

### Cost model (keep these in working memory)
| Operation | Cost | Note |
|---|---|---|
| `A @ B`, (m,k)@(k,n) | O(mkn) | order of a chain matters: associativity is free performance |
| `solve(A, b)`, dense n×n | ~n³/3 (Cholesky) to ~2n³/3 (LU) | one factorization, then n² per extra RHS |
| Full SVD / eigh, n×n | O(n³), constant ~10× a matmul | the expensive hammer; `svds`/`eigsh` for top-k |
| QR, m×n (m ≥ n) | O(mn²) | least squares without squaring κ |
| `pinv`, `matrix_rank`, `cond` | full SVD inside | never in a loop |
| Matrix–vector | O(n²) | keep matrices as operators (`LinearOperator`) when you only ever multiply |
Memory: an n×n fp64 matrix is 8n² bytes — n = 50,000 is 20 GB. If a "kernel matrix" or "attention matrix" appears in your plan, compute its memory before its math.

### Which norm?
- **L2 (vector)**: rotation-invariant; smooth; gradients linear. Default regularizer for shrinkage without sparsity.
- **L1**: ball corners sit on axes → sparsity at the optimum. Non-smooth at 0 — use proximal methods/coordinate descent; plain GD hovers near zero without hitting it.
- **L∞**: worst-case/adversarial budgets and error guarantees.
- **Spectral norm** (σ₁): operator amplification — "how much can this layer stretch a vector"; controls Lipschitz constants; the right norm for stability claims about maps.
- **Frobenius**: the matrix flattened to a vector — right for "total energy" and approximation error, wrong for operator behavior. ‖A‖_F² = Σσᵢ².
- Rule: quantity describes a *map* → spectral; describes *content* → Frobenius/L2. If a bound must hold for every input direction, it's spectral by definition.

### Sparse vs dense vs implicit
- Below ~95% zeros, sparse formats usually *lose* to dense BLAS — the indexing overhead swamps the flop savings. Measure before converting.
- Choose the sparse format by access pattern: CSR for row slicing and matvecs, CSC for column ops, COO/LIL only for construction — and never grow a CSR incrementally (each insert is O(nnz)).
- If you only ever *apply* the operator, don't materialize it at all: `scipy.sparse.linalg.LinearOperator(shape, matvec=f)` feeds directly into `eigsh`/`svds`/`cg`. Kernels, Hessians (via Hessian-vector products from autodiff), and graph Laplacians rarely need to exist as arrays.
- Iterative solvers (`cg` for SPD, `gmres` general) beat direct factorization when the matrix is large+sparse and you can tolerate a residual tolerance; their iteration count scales with √κ (CG) — preconditioning is not optional for κ ≫ 10⁴.

### Condition number rules
- κ(A) = σ₁/σₙ. Rough rule: a backward-stable solve loses about log₁₀(κ) decimal digits. κ = 10⁸ in float64 leaves ~8 digits (fine); in float32 leaves ~0 (garbage).
- If κ > 1/√ε for your dtype, act: ridge (adds λ to every σᵢ², capping effective κ at ~σ₁²/λ), truncate small singular values, or rescale features (most horrific κ values are just unit mismatch — meters vs micrometers — and vanish after standardization).
- κ is a property of the *problem*, not the algorithm. If κ·ε already explains your error, changing solvers won't help; reformulating (centering, scaling, orthogonalizing features) will.

### Exploit structure before reaching for generic routines
- **Diagonal / scaling**: never materialize `np.diag(d) @ A`; write `d[:, None] * A` (O(n²) → same, but no O(n²) memory for the diagonal matrix and no O(n³) matmul).
- **Low-rank update** (A + UVᵀ with U,V thin): Woodbury identity solves in O(n²k) instead of O(n³): (A + UVᵀ)⁻¹ = A⁻¹ − A⁻¹U(I + VᵀA⁻¹U)⁻¹VᵀA⁻¹. This is the trick behind efficient Kalman updates, L-BFGS, and rank-one Bayesian updates.
- **Kronecker / batched structure**: (A ⊗ B)vec(X) = vec(BXAᵀ) — never build the Kronecker product.
- **Symmetric PSD known by construction** (Gram matrices, covariances): use Cholesky/`eigh` pathways and add `1e-9 * tr(A)/n * I` jitter before factoring if it's PSD-by-math but indefinite-by-rounding.

## Failure modes & pitfalls

- **Silent broadcasting producing wrong-but-running code.** `(n,) - (n,1)` yields `(n,n)`, not an elementwise difference. Classic: `y_pred - y_true` where one is `(n,)` and the other `(n,1)` — the loss computes, the model trains, the numbers are nonsense. Correction: assert shapes at function boundaries (`assert y_pred.shape == y_true.shape`) or use einsum/einops where every axis is named.
- **`np.dot` vs `@` on >2-D arrays.** `np.dot(A, B)` on 3-D arrays sum-products the last axis of A against the *second-to-last* of B, producing an outer-batched monster; `A @ B` does sane batched matmul (broadcasting leading dims). For anything batched, write the einsum and stop guessing.
- **Forming AᵀA to solve least squares.** κ(AᵀA) = κ(A)². Normal equations turn a κ=10⁴ problem (fine in fp64) into κ=10⁸ (marginal in fp32, wasteful in fp64). Use QR/SVD-based `lstsq`. Same trap in disguise: computing a covariance as `X.T @ X / n` and eigendecomposing it, instead of taking the SVD of centered X directly — the singular values are the square roots you wanted, at double the accuracy.
- **PCA without centering.** SVD of uncentered data points the first component at the mean, not the max-variance direction. Always subtract `X.mean(axis=0)`. Standardizing columns too is a modeling choice: do it when features have incomparable units, skip it when relative scale is meaningful.
- **Using `eig` on a symmetric matrix.** Round-off asymmetry yields eigenvalues like `2.9999999+1e-16j`, then `.real` gets sprinkled around as a "fix". Correction: symmetrize `(A + A.T)/2`, call `eigh`.
- **Comparing eigenvectors/singular vectors across runs or libraries and reporting a "bug".** They're defined up to sign, and up to arbitrary rotation within a repeated-eigenvalue subspace. Compare `min(‖v1−v2‖, ‖v1+v2‖)`, or compare projectors `v vᵀ` / subspace principal angles — never raw vectors.
- **`full_matrices=True` (NumPy's default) on tall matrices.** For a 10⁶×100 matrix it allocates a 10⁶×10⁶ U. Pass `full_matrices=False` (economy SVD) unless you specifically need the full orthonormal basis of the ambient space.
- **Treating `pinv` as free.** `np.linalg.pinv` runs a full SVD, O(mn·min(m,n)). Inside a loop it is often the entire runtime. Factor once outside the loop, or use `lstsq` per right-hand side.
- **Learning-rate mysticism that is an eigenvalue fact.** For quadratic loss ½xᵀHx, GD is x ← (I − ηH)x; convergence needs every |1 − ηλᵢ| < 1 ⟹ η < 2/λmax. Oscillation/explosion along one direction while flat directions crawl is κ(H) in action — this is *why* preconditioning, normalization layers, and adaptive optimizers exist. Diagnose "training is unstable" by asking about the sharpest curvature, not by folklore.
- **`np.linalg.matrix_rank` default tolerance on mixed-scale data.** The default tol scales with σ₁ and matrix size — usually right, but if columns span 1e-6..1e6, genuine directions get zeroed. Scale columns first or pass explicit `tol`.
- **einsum index slips.** An index appearing in inputs but not the output is *summed*: `np.einsum('ij,ij->', A, B)` is a scalar (full contraction); `'ij,ij->ij'` is elementwise. Always write the explicit `->` output. For chains, pass `optimize=True` (or use `opt_einsum`) — contraction *order* changes complexity class: `(A@B)@v` is O(n³) while `A@(B@v)` is O(n²) for the same expression.
- **Checking PD with eigenvalues > 0 after roundoff.** A covariance assembled in fp32 routinely has eigenvalues like −3e-9. Don't reject; don't `abs()` them either — clip at 0 or add jitter, and know which one your downstream math needs (sampling needs PD, so jitter).
- **Solving with a matrix you built from differences.** Finite-difference or near-duplicate feature columns → near-singular A. The symptom is a huge-norm solution with a small residual. Detect via κ or tiny σₙ, then regularize or drop the redundant column — do not just switch solvers.
- **Batched code that mixes batch and feature axes in a reshape.** `x.reshape(B, -1)` after a transpose that wasn't done (or was done twice) interleaves samples. After any reshape/transpose chain, verify with a tiny tensor whose entries encode their own coordinates (`np.arange(24).reshape(2,3,4)`) that element (b, i, j) lands where you think.

- **Confusing "orthogonal projection" with "normalize then subtract".** Projection onto the span of non-orthonormal columns W is `W @ solve(W.T @ W, W.T @ y)` (or QR of W) — not `W @ (W.T @ y)`, which is only valid when WᵀW = I. Symptom: "removing" a direction leaves a residual correlated with it.
- **Gram matrix vs covariance confusion in kernel/feature code.** `X @ X.T` is (n,n) sample-similarity; `X.T @ X` is (p,p) feature-covariance-ish. Both appear in PCA derivations (dual vs primal); picking the smaller one is a legitimate O(min(n,p)²·max) optimization, but mixing up which eigenvectors need the `X`-multiplication to convert (v = Xᵀu/σ) produces subtly wrong components.

## Worked micro-examples

**1. Attention scores in einsum with shape discipline.**
```python
# q: (B, H, T, d)  k: (B, H, S, d)  v: (B, H, S, dv)
scores = np.einsum('bhtd,bhsd->bhts', q, k) / np.sqrt(d)   # (B, H, T, S)
attn   = softmax(scores, axis=-1)                          # each query's row over S sums to 1
out    = np.einsum('bhts,bhsv->bhtv', attn, v)             # (B, H, T, dv)
```
Every index appears where it should, and the softmax axis is the *summed-over key* axis. Had you written `'bhtd,bhsd->bhst'`, the later `softmax(..., axis=-1)` would normalize over queries — a bug that still trains, just badly.

**2. Ridge regression via SVD shows what regularization does spectrally.**
```python
U, s, Vt = np.linalg.svd(X, full_matrices=False)      # X: (n, p), centered
# OLS:   w = V diag(1/s) Uᵀ y         — blows up where s is small
# Ridge: w = V diag(s/(s²+λ)) Uᵀ y    — filter factor s²/(s²+λ)
w_ridge = Vt.T @ ((s / (s**2 + lam)) * (U.T @ y))
```
Directions with s ≫ √λ pass untouched; directions with s ≪ √λ are suppressed. Ridge is a smooth truncated SVD, and the entire "regularize more when data is noisy" intuition is this spectral filter. Bonus: this form gives ridge for *all* λ values from one SVD — the right way to sweep a regularization path.

**3. GD step-size limit from Hessian eigenvalues, with numbers.**
Quadratic with H = diag(100, 1): λmax=100, λmin=1, κ=100. Stability needs η < 2/100 = 0.02. At η = 0.019: the fast direction contracts by |1 − 1.9| = 0.9 per step (ricocheting sign each step); the slow one by 1 − 0.019 = 0.981 → ~120 steps per e-fold. The LR is pinned by λmax while progress is set by λmin — that gap *is* κ. Newton (multiply the gradient by H⁻¹) makes both directions converge in one step: second-order and adaptive methods attack the ratio, not the scale.

**4. Woodbury in anger: rank-k update of a solved system.**
You've factored A (n = 10⁴, Cholesky ~ n³/3 ≈ 3×10¹¹ flops) and now need (A + uuᵀ)⁻¹b for a rank-1 update u. Refactoring costs another n³/3. Woodbury:
```python
Ainv_b = cho_solve(cA, b)        # O(n²) with the existing factor
Ainv_u = cho_solve(cA, u)        # O(n²)
x = Ainv_b - Ainv_u * (u @ Ainv_b) / (1.0 + u @ Ainv_u)
```
O(n²) instead of O(n³) — a 10⁴× saving at this size, and the pattern behind Kalman filters, online ridge updates, and L-BFGS's implicit Hessian. Numerical caveat: repeated low-rank updates accumulate error; refactor from scratch every few hundred updates.

**5. Power iteration for the top eigenpair — and why it can stall.**
```python
v = rng.normal(size=n); v /= np.linalg.norm(v)
for _ in range(iters):
    v = A @ v
    v /= np.linalg.norm(v)
lam = v @ A @ v
```
Convergence rate is |λ₂/λ₁| per iteration — a spectral *gap* fact. λ₂/λ₁ = 0.99 needs ~100 iterations per digit; that's why practical top-k code uses Lanczos (`eigsh`) or subspace iteration with orthogonalization, and why "power iteration is slow" usually means "your spectrum is flat".

## Verification & self-check

- **Shape audit**: state the shape of every intermediate; derive the output shape from the einsum string alone before trusting any values.
- **Reconstruction check** after any decomposition: `‖A − U @ np.diag(s) @ Vt‖ / ‖A‖ < 1e-10` (fp64). Catches transposes, ordering, and economy-vs-full mistakes in one line.
- **Solve check**: report the relative residual `‖Ax−b‖ / (‖A‖·‖x‖ + ‖b‖)`, not "it ran". Small residual + huge ‖x‖ = ill-conditioning; say so.
- **Orthogonality check**: `‖QᵀQ − I‖` for anything claimed orthonormal; re-orthogonalize if it drifts above ~1e-10 in long iterations.
- **Tiny-case oracle**: run the routine on a 3×3 diagonal or rank-1 matrix with a hand-computable answer before trusting real shapes.
- **Invariance tests**: PCA must be invariant to row permutation; least-squares solutions must be invariant to duplicating a (row, target) pair with half weight; predictions must be invariant to feature reordering when weights are reordered too. A violated invariance localizes the bug to centering, weighting, or broadcasting.
- **Spectrum eyeball**: before claiming rank/conditioning conclusions, print the singular values (log scale). A clean gap → trustworthy rank; a smooth decay → "rank" is a modeling choice, present it as such.
- **Gradient check for anything hand-derived**: matrix-calculus derivations (∂/∂W of losses, backward passes) verified against central finite differences at a random point, rtol ≤ 1e-5 in fp64. The most common analytic errors are a dropped transpose and a factor of 2 from symmetric terms — both invisible until checked.
- **Units/scale audit**: if κ looks catastrophic, check whether two features differ by a unit conversion factor (1e3, 1e6) before reaching for fancier decompositions; standardization fixes most "numerically hard" regressions outright.
