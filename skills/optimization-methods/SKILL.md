---
name: optimization-methods
description: Loads when solving or debugging optimization problems — choosing between gradient methods, LP/QP/MIP solvers, and heuristics; diagnosing slow/diverging training; applying Lagrangians/KKT; tuning Adam vs SGD; recognizing convexity or NP-hard structure; or reformulating a problem (relaxation, change of variables, dual) to make it tractable.
---

# Optimization Across Math and ML

## Core mental model

1. **Reformulation beats algorithm choice.** The top expert move is changing the problem: log-transform to make a product objective additive, substitute variables to remove a constraint, relax integrality then round, take the dual when constraints outnumber variables, add slack variables to convert max/abs into linear constraints. An hour of reformulation routinely beats a week of solver tuning.
2. **Convexity is a certificate, not a vibe.** If the problem is convex, any local minimum is global, duality is (usually) tight, and off-the-shelf solvers give reliable answers with optimality guarantees. Learn to *recognize* convexity compositionally (the DCP rules): affine ∘ convex is convex; max of convex is convex; nonnegative sums preserve it; log/exp/norms/quadratics have known curvature. If you can express it in CVXPY without errors, it's convex — that's a practical test, not a metaphor.
3. **Condition number is the speed limit of first-order methods.** On a quadratic with Hessian eigenvalues in [μ, L], gradient descent with optimal step converges like ((κ−1)/(κ+1))^t, κ = L/μ. κ = 10⁴ means ~5000 iterations per digit. Everything in the modern toolkit — momentum (√κ dependence), preconditioning, Adam's per-coordinate scaling, batch/layer norm — is an attack on κ.
4. **In high dimensions, saddle points, not local minima, are the obstacle.** For random-ish landscapes, critical points with high loss are overwhelmingly saddles (some negative curvature to escape); poor local minima are rarer than folklore says, which is why plain SGD + noise works on ugly non-convex problems. Diagnose "stuck" as: saddle/plateau (gradient tiny, loss mediocre → add noise/momentum, check init) before assuming "bad local min".
5. **Structure determines the tool.** Smooth + unconstrained → quasi-Newton (L-BFGS) or GD-family. Linear/quadratic + constraints → LP/QP solver. Integer decisions → MIP or accept heuristics. Black-box, expensive, low-dim → Bayesian optimization. Black-box, cheap, weird → CMA-ES/simulated annealing. Reaching for gradient descent on everything is the generalist tell.

## Decision frameworks

### Solver selection table
| Problem shape | Tool | Notes |
|---|---|---|
| Smooth, deterministic, ≤ ~10⁶ vars, full gradients affordable | L-BFGS (`scipy.optimize.minimize(method='L-BFGS-B')`) | Superlinear near optimum; almost always beats hand-tuned GD on deterministic objectives |
| Stochastic objective (minibatch losses) | SGD+momentum or Adam family | L-BFGS breaks under gradient noise; line searches lie |
| Linear objective + linear constraints | LP solver (HiGHS via `scipy.optimize.linprog(method='highs')`) | Exact optimum in seconds up to millions of vars; do not gradient-descend an LP |
| Convex quadratic + constraints | QP/conic solver (OSQP, Clarabel; model in CVXPY) | Portfolio, SVM-like, MPC problems |
| Discrete choices, need quality guarantees, ≤ ~10⁵ binaries | MIP (HiGHS, CBC; OR-Tools CP-SAT for scheduling/logic) | Branch-and-bound gives an optimality *gap* — you know how far off you are |
| Discrete, huge, time-boxed | Heuristics: local search, LNS, greedy + 2-opt; keep the MIP as a small-instance oracle | Validate heuristic on instances small enough for exact solve |
| Black-box, expensive evaluations (< ~1000 evals), ≤ ~20 dims | Bayesian optimization (Optuna, botorch) | Hyperparameters, simulations |
| Black-box, cheap, rugged, ≤ ~100 dims | CMA-ES (`cma` package) | Shockingly strong default; beats random/anneal in most benchmarks |
| Nonlinear least squares specifically | Levenberg–Marquardt (`scipy.optimize.least_squares`) | Exploits residual structure; never use generic minimize for curve fitting |

### Adam vs SGD — the evidence-based rules
- **Default Adam(W)** for: transformers/attention, sparse gradients (embeddings), anything with heterogeneous per-parameter gradient scales, and any situation where you can't afford an LR search. Its per-coordinate normalization is a cheap diagonal preconditioner.
- **SGD+momentum can match or beat Adam** mainly on vision-style convnets with well-tuned LR schedules; its advantage there is small and costs tuning effort. The "Adam generalizes worse" folklore largely dissolved once weight decay was fixed: use **AdamW** (decoupled decay), because in plain Adam, L2 penalty gets divided by the second-moment estimate and effectively vanishes for high-gradient weights.
- Adam's ε is a real hyperparameter, not a formality: with tiny second moments, effective LR ≈ lr/ε-scaled; raising ε (1e-8 → 1e-4) tames spikes in low-signal regimes.
- Learning-rate warmup matters for Adam because early second-moment estimates are noisy; a few hundred–few thousand warmup steps is standard, not superstition.

### Step size: line search vs fixed
- Deterministic full-batch objective → always line search (Armijo/Wolfe; built into L-BFGS). Hand-picking a step for a deterministic problem is leaving free accuracy on the table.
- Stochastic minibatch → fixed schedule (cosine/linear decay + warmup). Line search on noisy gradients chases noise. If loss diverges immediately: LR above 2/L for the sharpest curvature — cut 10×. If loss plateaus early and high: usually too *low* an LR or bad conditioning, not a "local minimum".
- Batch size tradeoff: larger batches reduce gradient variance (allowing ~proportionally larger LR up to a critical batch size) but cost linearly more compute per step; past the critical size you pay compute for no optimization speedup. Small batches also act as implicit regularization noise.

### Constrained problems — the KKT ladder
1. Try to **eliminate** the constraint: reparameterize (x = exp(z) for x>0; softmax for simplex; x = l + (u−l)·sigmoid(z) for boxes).
2. Simple set you can project onto (box, ball, simplex) → **projected gradient**.
3. General smooth constraints → model it in CVXPY if convex; otherwise SLSQP/IPOPT-class solvers.
4. **Penalties last, and start soft**: a huge penalty weight from step one wrecks conditioning (κ scales with the penalty weight); use augmented Lagrangian or increase the weight on a schedule.
5. Read the multipliers: λᵢ is the objective's sensitivity to relaxing constraint i (shadow price). λᵢ = 0 → constraint inactive, drop it and re-solve to simplify; huge λᵢ → that constraint is what's costing you, negotiate *it*.

### NP-hard recognition reflex
When a subproblem smells like subset selection, assignment with conflicts, routing, coloring, or "best subset of features/items under a budget with interactions": stop searching for a poly-time exact trick. Choose deliberately among (a) MIP with a time limit — take the incumbent + gap; (b) convex relaxation (L1 for L0/cardinality, SDP/spectral for combinatorial) then round; (c) greedy with a guarantee — if the objective is monotone submodular (coverage, diminishing returns), greedy is within (1−1/e) ≈ 0.63 of optimal, which is usually the correct answer to "select k items"; (d) domain heuristic + evaluation harness. Saying "here's an efficient exact algorithm" for an NP-hard structure is a critical failure.

## Failure modes & pitfalls

- **Gradient-descending a problem a solver eats for breakfast.** Assignment problems (`scipy.optimize.linear_sum_assignment` — Hungarian, exact, fast), LPs, small QPs, and isotonic regression all have exact solvers. GD gives you approximate answers with tuning pain for problems with exact poly-time solutions.
- **Plain Adam + L2 regularization believing it decays weights.** It mostly doesn't (see above). Use AdamW; in PyTorch that's `torch.optim.AdamW`, and note `weight_decay` in `torch.optim.Adam` is the broken coupled version.
- **Penalty method with λ=1e9 from the start.** The Hessian gains eigenvalues of order λ → κ explodes → GD stalls or oscillates. Ramp the penalty, or use augmented Lagrangian (adds a multiplier estimate so moderate λ suffices).
- **Declaring convexity from a plot, or "the sum of two convex functions' ratio".** Ratios, products, and differences of convex functions are generally *not* convex (x² − x⁴, x/y). Verify by DCP composition rules or a failed CVXPY formulation. Conversely, don't miss hidden convexity: geometric programs (posynomials) become convex after log-log transform.
- **Trusting `scipy.optimize.minimize` defaults blindly.** Default method (BFGS) with numerically-differenced gradients on a noisy objective produces garbage silently — check `result.success`, `result.message`, and gradient norm at the "solution". If evaluations are noisy, finite differences are meaningless; switch to Nelder-Mead/CMA-ES or fix the noise.
- **One local optimum = the answer, for multimodal problems.** Multistart is not optional for non-convex fits (mixture models, MLE with multiple modes, neural-net-free but non-convex objectives): 10–50 random restarts, keep the best, and *report* the spread — a wide spread is diagnostic information.
- **Normalizing inputs is treated as preprocessing trivia; it's conditioning.** Unscaled features with ranges 1 vs 10⁶ produce κ ≥ 10¹² on a linear model. No optimizer setting rescues this; standardize (or whiten) and the "hard" optimization vanishes. First question for any slow convergence: what's the scale spread of inputs/parameters?
- **Confusing "loss stopped improving" with convergence.** Check the gradient norm. Tiny gradient + mediocre loss = plateau/saddle (fix: momentum, LR bump, better init). Large oscillating gradient = LR too high or batch too small. Loss improving on train but not eval is not an optimization problem at all — stop tuning the optimizer.
- **Early stopping on the *test* metric, tuning LR on test, etc.** — optimization decisions made on held-out data quietly become training. Keep a validation split for all optimizer/schedule choices.
- **Duality misuse:** taking the dual and forgetting to check strong duality conditions (convex + Slater's condition: a strictly feasible point). For non-convex problems the dual gives a *bound*, not the answer — valuable (e.g., for branch-and-bound and Lagrangian relaxation), but don't report the dual optimum as the solution.
- **Ignoring the incumbent/gap semantics of MIP solvers.** A MIP hitting its time limit returns the best feasible solution *plus a bound*. "Solver timed out" is not failure — report "within 2.3% of optimal, proven". Set `mip_rel_gap` deliberately rather than waiting hours for the last 0.1%.

## Worked micro-examples

**1. Reformulation: L1 regression as an LP (no subgradient hacks).**
min ‖Ax − b‖₁ becomes: introduce t ∈ ℝᵐ, minimize Σtᵢ s.t. −t ≤ Ax − b ≤ t.
```python
import cvxpy as cp
x = cp.Variable(n)
prob = cp.Problem(cp.Minimize(cp.norm1(A @ x - b)))
prob.solve()          # CVXPY does the LP reformulation internally — exact, fast
```
The same slack trick converts minimax (min max_i fᵢ) and any |·| in objectives/constraints into linear form. Recognizing "this nonsmooth thing is secretly an LP" is worth more than any subgradient schedule.

**2. Condition number → concrete step-count arithmetic.**
f(x) = ½(x₁² + 100x₂²): L = 100, μ = 1, κ = 100. Best fixed step η* = 2/(L+μ) ≈ 0.0198; contraction factor (κ−1)/(κ+1) ≈ 0.98 per step → ~115 steps per e-fold, ~1150 steps for 10⁻⁴ error. Heavy-ball momentum: factor (√κ−1)/(√κ+1) ≈ 0.82 → ~12 steps per e-fold, ~10× fewer. Rescale x₂' = 10·x₂ and κ = 1: one Newton-like step. Same problem, three costs — conditioning is the variable.

**3. KKT reading on a budget allocation.**
max Σᵢ αᵢ log(xᵢ) s.t. Σxᵢ = B, xᵢ ≥ 0. Lagrangian stationarity: αᵢ/xᵢ = λ → xᵢ = αᵢ/λ; the budget gives λ = Σαⱼ/B, so xᵢ = B·αᵢ/Σαⱼ — proportional allocation, derived not guessed. The multiplier λ = Σαⱼ/B is the marginal utility of budget: doubling B halves λ, quantifying exactly how much another dollar of budget is worth. Log-utility + linear budget → proportional split is a pattern worth caching (it's also why softmax-style allocations keep reappearing).

## Verification & self-check

- **Check optimality certificates, not just objective values**: gradient norm ≈ 0 (unconstrained), KKT residuals + complementary slackness (constrained), duality gap (convex solvers report it), MIP gap (branch-and-bound).
- **Perturb the "optimum"**: evaluate f at x* ± small random steps; if anything is lower, you weren't done. Cheap and catches premature-stop bugs.
- **Verify the gradient itself** when using custom gradients: compare to central finite differences at a random point (`scipy.optimize.check_grad`); relative error should be ≲1e-6 in float64. Most "optimizer isn't working" reports are wrong gradients.
- **Feasibility first**: for constrained answers, plug into every constraint with tolerances and report max violation; solvers' "optimal" can hide 1e-4 violations that matter downstream.
- **Cross-check with a second method on a shrunk instance**: exact/brute-force on n=10 vs your heuristic; solver A vs solver B. Disagreement in the objective beyond tolerance means a modeling bug, usually a sign flip or missing constraint.
- **Restart audit for non-convex claims**: if you're asserting "the optimum is X", show the multistart spread that supports the claim.
