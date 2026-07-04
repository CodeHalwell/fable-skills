---
name: linear-algebra-for-ml
description: Loads when working with matrix computations in ML contexts — debugging shape/broadcasting errors, choosing decompositions (SVD/eig/QR/Cholesky), reasoning about conditioning, curvature, or learning rates, writing einsum, implementing PCA/least-squares/attention, or deciding whether to invert, solve, or factor.
---

# Linear Algebra as Practiced in ML

## Core mental model

1. **Shapes are the type system.** Every matrix bug is a shape bug you haven't found yet. Before writing any tensor expression, write the shape signature as a comment: `(B, T, d) @ (d, k) -> (B, T, k)`. Dimension bookkeeping catches ~80% of bugs before the first run — silent broadcasting catches the rest of you.
2. **SVD is the master decomposition.** `A = U Σ Vᵀ` gives you, for free: rank (count of nonzero σᵢ), best low-rank approximation (truncate — Eckart–Young says this is optimal in both Frobenius and spectral norm), PCA (SVD of centered data), pseudoinverse (`V Σ⁺ Uᵀ`), condition number (σ₁/σᵣ), null space and range (columns of V and U). When unsure which tool applies, ask "what does the SVD of this matrix look like?"
3. **Eigenvalues govern dynamics.** Anything iterated — power iteration, gradient descent, RNN state, Markov chains — is controlled by the spectrum of the iteration matrix. |λ|max < 1 means contraction/stability; the ratio of extreme eigenvalues means slow directions. Hessian eigenvalues are curvature: GD diverges when lr > 2/λmax, and converges at a rate set by κ = λmax/λmin.
4. **Never invert; solve or factor.** `inv(A) @ b` is slower, less accurate, and destroys structure (sparsity, symmetry). `solve(A, b)` uses a factorization directly. Explicit inverses are justified almost only when you need many entries of the inverse itself (e.g., full covariance of an estimator).
5. **Numerical rank ≠ mathematical rank.** In floating point, ask "how many singular values exceed tolerance?" not "is the determinant zero?" Determinants are useless as singularity tests (`det(0.1 * I_300)` underflows to 0 for a perfectly well-conditioned matrix).

## Decision frameworks

### Which factorization?
| Situation | Use | Why |
|---|---|---|
| Solve Ax=b, A symmetric positive definite | Cholesky (`cho_factor`/`cho_solve`) | 2x faster than LU, fails loudly if not PD (a free PD test) |
| Solve Ax=b, general square A | LU via `np.linalg.solve` | Standard; backward stable with pivoting |
| Least squares, well-conditioned, m≫n | QR (`np.linalg.lstsq` uses SVD; `scipy.linalg.lstsq` with `lapack_driver='gelsy'` uses QR) | Never form AᵀA — that squares the condition number |
| Least squares, rank-deficient or ill-conditioned | SVD with singular-value cutoff, or ridge | Pseudoinverse handles rank deficiency explicitly |
| Need top-k eigen/singular pairs of a huge matrix | Iterative: `scipy.sparse.linalg.eigsh`/`svds`, or randomized SVD (`sklearn.utils.extmath.randomized_svd`) | Full decomposition is O(n³); you only need k directions |
| Symmetric/Hermitian eigenproblem | `eigh`, never `eig` | `eigh` is faster, returns real eigenvalues in sorted order, and won't hand you spurious tiny imaginary parts |
| Repeated solves with same A, different b | Factor once (`lu_factor`, `cho_factor`), solve many | Factorization is O(n³); each solve is O(n²) |

### Which norm?
- **L2**: rotation-invariant; gradients are linear; smooth everywhere. Default for regularization when you want shrinkage without sparsity.
- **L1**: corners of the ball sit on axes → sparsity. Non-smooth at 0, so use proximal methods or subgradients, not plain GD claims of convergence.
- **L∞**: adversarial-perturbation budgets, worst-case error bounds.
- **Spectral norm** (largest σ): operator amplification — "how much can this layer stretch a vector"; bounds Lipschitz constants of linear layers.
- **Frobenius norm**: treat the matrix as a flat vector; right for "total energy" and low-rank approximation error, wrong for operator behavior.
- Rule: if the quantity describes a *map*, use spectral; if it describes *content*, use Frobenius/L2.

### Condition number decision rule
κ(A) = σ₁/σₙ. Rough rule: you lose log₁₀(κ) decimal digits of accuracy in a solve. κ ≈ 10⁸ in float64 leaves ~8 digits (fine); the same in float32 leaves ~0 digits (garbage). If κ > 1/√ε for your dtype, regularize (ridge: adds λ to every σᵢ², capping effective κ) or truncate small singular values.

## Failure modes & pitfalls

- **Silent broadcasting producing wrong-but-running code.** `(n,) - (n,1)` yields `(n,n)`, not elementwise difference. Classic: `y_pred - y_true` where one is shape `(n,)` and the other `(n,1)` — the loss computes, trains, and is nonsense. Correction: assert shapes at function boundaries (`assert y_pred.shape == y_true.shape`) or use `einsum`/`einops` where shapes are explicit.
- **`np.dot` / `@` on >2-D arrays behaving differently.** `np.dot(A, B)` on 3-D arrays does a sum-product over the last axis of A and second-to-last of B (an outer batching you almost never want); `A @ B` does batched matmul. For anything batched, write the einsum.
- **Forming AᵀA to solve least squares.** κ(AᵀA) = κ(A)². Normal equations turn a κ=10⁴ problem (fine) into κ=10⁸ (marginal). Use QR/SVD-based `lstsq`. Same trap wearing a different hat: computing covariance as `X.T @ X / n` then eigendecomposing, instead of taking SVD of centered X directly.
- **PCA without centering.** SVD of uncentered data makes the first component point at the mean, not the direction of maximum variance. Always `X - X.mean(axis=0)` first. (Whether to also standardize columns is a modeling choice: do it when features have incomparable units.)
- **Using `eig` on a symmetric matrix and getting complex dust.** Round-off asymmetry gives eigenvalues like `2.9999999+1e-16j`. Then `.real` gets sprinkled around as a "fix". Correction: symmetrize (`(A + A.T)/2`) and call `eigh`.
- **Comparing eigenvectors across libraries/runs and finding "bugs".** Eigenvectors are defined up to sign (and arbitrary rotation within a repeated eigenvalue's subspace). Test `min(‖v1-v2‖, ‖v1+v2‖)` or compare projectors `v vᵀ`, never raw vectors.
- **Expecting `svd` U,V signs to be deterministic.** Same issue; also full_matrices=True by default in NumPy returns huge U for tall matrices — pass `full_matrices=False` for the economy SVD unless you need the full orthonormal basis.
- **Treating `pinv` as free.** `np.linalg.pinv` runs a full SVD — O(mn·min(m,n)). Inside a loop it's often the entire runtime. Factor once outside the loop, or use `lstsq`.
- **Learning-rate confusion that is really an eigenvalue fact.** For quadratic loss ½xᵀHx, GD update is x ← (I − ηH)x; it converges iff every |1−ηλᵢ| < 1, i.e., η < 2/λmax. Loss oscillating/exploding in the sharpest direction while barely moving in flat directions is κ(H) at work — this is *why* preconditioning, normalization, and adaptive methods exist, not folklore.
- **Rank checks via `np.linalg.matrix_rank` with default tolerance on badly scaled data.** The default tol scales with σ₁ and matrix size, which is usually right — but if your matrix mixes scales (some columns ~1e6, some ~1e-6), scale columns first or pass an explicit tol; otherwise real directions get zeroed.
- **einsum mistakes:** repeating an index you meant to keep sums it away. `np.einsum('ij,ij->', A, B)` is a full contraction (scalar); `'ij,ij->ij'` is elementwise. Write the output indices explicitly, never rely on implicit mode. For chained contractions pass `optimize=True` (or use `opt_einsum`) — contraction order changes complexity, e.g. `(A@B)@v` is O(n³) but `A@(B@v)` is O(n²).

## Worked micro-examples

**1. Attention scores in einsum with shape discipline.**
```python
# q: (B, H, T, d)  k: (B, H, S, d)  v: (B, H, S, dv)
scores = np.einsum('bhtd,bhsd->bhts', q, k) / np.sqrt(d)   # (B, H, T, S)
attn   = softmax(scores, axis=-1)                          # rows over S sum to 1
out    = np.einsum('bhts,bhsv->bhtv', attn, v)             # (B, H, T, dv)
```
Every index appears where it should; the softmax axis is the *summed-over key* axis. If you had written `'bhtd,bhsd->bhst'` the later softmax over `-1` would normalize over queries — a real bug seen in the wild that still "trains".

**2. Ridge regression via SVD shows what regularization does spectrally.**
```python
U, s, Vt = np.linalg.svd(X, full_matrices=False)      # X: (n, p), centered
# OLS:   w = V diag(1/s) Uᵀ y        — blows up small s
# Ridge: w = V diag(s/(s²+λ)) Uᵀ y   — filter factor s²/(s²+λ)
w_ridge = Vt.T @ ((s / (s**2 + lam)) * (U.T @ y))
```
Directions with s ≫ √λ are untouched; directions with s ≪ √λ are suppressed. Ridge = smooth truncated SVD. Choosing λ ≈ σ_noise² / σ_signal² is the spectral picture behind "regularize more when noisy".

**3. GD step-size limit from Hessian eigenvalues, with numbers.**
Quadratic with H = diag(100, 1): λmax=100, λmin=1, κ=100. Stability requires η < 2/100 = 0.02. At η=0.019, the slow direction contracts by factor (1−0.019·1)=0.981 per step → ~120 steps per e-fold of error; the fast direction ricochets at −0.9. Momentum/precise preconditioning (multiply by H⁻¹ ≈ Newton) fixes the *ratio*, not the scale — that is the whole game of second-order and adaptive methods.

## Verification & self-check

- **Shape audit**: state the shape of every intermediate; confirm the output shape from the einsum string alone before trusting values.
- **Reconstruction check**: after any decomposition, verify `‖A − U @ np.diag(s) @ Vt‖ / ‖A‖ < 1e-10` (float64). Cheap and catches transposition/ordering errors.
- **Solve check**: report the relative residual `‖Ax−b‖/(‖A‖‖x‖ + ‖b‖)`, not just "it ran". A small residual with a huge ‖x‖ signals ill-conditioning.
- **Orthogonality check**: `‖QᵀQ − I‖` for anything claimed orthonormal.
- **Tiny-case oracle**: run the routine on a 3×3 with a hand-computable answer (e.g., diagonal or rank-1 matrix) before trusting it on real shapes.
- **Invariance tests**: PCA output shouldn't change under row permutation; least-squares solution shouldn't change if you duplicate a data point *and* its target consistently... if it does, you have a centering, weighting, or broadcasting bug.
