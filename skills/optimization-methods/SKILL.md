---
name: optimization-methods
description: Loads when solving or debugging optimization problems — choosing between gradient methods, LP/QP/MIP solvers, and heuristics; diagnosing slow/diverging training; applying Lagrangians/KKT; tuning Adam vs SGD; recognizing convexity or NP-hard structure; or reformulating a problem (relaxation, change of variables, dual) to make it tractable.
---

# Optimization Across Math and ML

## Core mental model

1. **Reformulation beats algorithm choice.** The top expert move is changing the problem: log-transform to make a product objective additive, substitute variables to eliminate a constraint, relax integrality then round, take the dual when constraints outnumber variables, introduce slacks to turn max/|·| into linear constraints. An hour of reformulation routinely beats a week of solver tuning.
2. **Convexity is a certificate, not a vibe.** Convex ⟹ every local minimum is global, duality is (usually) tight, and off-the-shelf solvers return answers with optimality guarantees. Recognize convexity *compositionally* (the DCP rules): affine ∘ convex is convex; max of convex is convex; nonnegative-weighted sums preserve it; norms, quadratics, exp, −log have known curvature. Practical test: if CVXPY accepts the formulation without a DCP error, it's convex.
3. **Condition number is the speed limit of first-order methods.** On a quadratic with Hessian eigenvalues in [μ, L], optimally-stepped GD converges like ((κ−1)/(κ+1))^t with κ = L/μ. κ = 10⁴ means thousands of iterations per digit. Everything in the modern toolkit — momentum (√κ dependence), preconditioning, Adam's per-coordinate scaling, batch/layer norm, feature standardization — is an attack on κ.
4. **In high dimensions, saddles and plateaus, not local minima, are the obstacle.** For random-ish landscapes, high-loss critical points are overwhelmingly saddles (with escape directions); genuinely bad local minima are rarer than folklore says — which is why SGD + noise works on ugly non-convex problems. Diagnose "stuck" as saddle/plateau (tiny gradient, mediocre loss → noise, momentum, better init) before blaming "local minima".
5. **Structure determines the tool.** Smooth + deterministic → L-BFGS. Stochastic minibatch → SGD/Adam family. Linear/quadratic + constraints → LP/QP solver. Integer decisions → MIP/CP-SAT or admit heuristics. Expensive black-box → Bayesian optimization. Cheap rugged black-box → CMA-ES. Reaching for gradient descent on everything is the generalist tell.
6. **Optimality is a checkable claim.** Gradient norms, KKT residuals, duality gaps, and MIP bounds are certificates. An answer without its certificate is a guess with confidence.

## Decision frameworks

### Solver selection table
| Problem shape | Tool | Notes |
|---|---|---|
| Smooth, deterministic, full gradients affordable | L-BFGS (`scipy.optimize.minimize(method='L-BFGS-B')`) | Superlinear near optimum; nearly always beats hand-tuned GD on deterministic objectives |
| Stochastic objective (minibatch losses) | SGD+momentum or AdamW | L-BFGS breaks under gradient noise; line searches chase noise |
| Linear objective + linear constraints | LP solver (HiGHS via `scipy.optimize.linprog(method='highs')`) | Exact optimum, huge scale; never gradient-descend an LP |
| Convex quadratic / conic + constraints | QP/conic solver (OSQP, Clarabel), modeled in CVXPY | Portfolio, SVM-style, MPC |
| Discrete choices, need quality guarantees | MIP (HiGHS, CBC, Gurobi) or CP-SAT (OR-Tools) for scheduling/logic | Branch-and-bound reports an optimality *gap* — you know how far off you are |
| Discrete, huge, time-boxed | Heuristics: greedy + local search, LNS; keep an exact solver as small-instance oracle | Validate the heuristic against exact solves on shrunk instances |
| Black-box, expensive (< ~10³ evals), ≤ ~20 dims | Bayesian optimization (Optuna, BoTorch) | Hyperparameters, simulator tuning |
| Black-box, cheap, rugged, ≤ ~100 dims | CMA-ES (`cma` package) | Strong default; usually beats annealing and random search |
| Nonlinear least squares | `scipy.optimize.least_squares` (Levenberg–Marquardt / TRF) | Exploits residual structure; never use generic `minimize` for curve fitting |
| Assignment / matching | `scipy.optimize.linear_sum_assignment` | Hungarian algorithm: exact, poly-time — not a search problem |

### Adam vs SGD — folklore vs evidence
- **Default AdamW** for: transformers/attention stacks, sparse gradients (embeddings), heterogeneous per-parameter gradient scales, and whenever you can't afford a careful LR search. Per-coordinate second-moment scaling is a cheap diagonal preconditioner.
- **SGD+momentum** can match or beat Adam mainly on vision-style convnets with well-tuned schedules; the edge is small and costs tuning. The "Adam generalizes worse" folklore largely dissolved once weight decay was fixed: in plain Adam, an L2 penalty gets divided by the second-moment estimate and effectively vanishes for high-gradient weights. Use decoupled decay — `torch.optim.AdamW` — and note that `weight_decay` in `torch.optim.Adam` is the broken coupled version.
- Adam's ε is a real hyperparameter: with tiny second moments the effective step scales like lr/√(v̂+ε); raising ε (1e-8 → 1e-4) damps spikes in low-signal regimes.
- Warmup for Adam is not superstition: early second-moment estimates are noisy, so the first steps are wildly mis-scaled without a few hundred–few thousand warmup steps.
- Gradient clipping (by global norm, e.g. 1.0) is cheap insurance against loss spikes from rare large gradients; it changes the direction only on the clipped steps.

### Step size: line search vs fixed schedule
- Deterministic full-batch → always line search (Armijo/Wolfe; built into L-BFGS/CG). Hand-picking a fixed step for a deterministic problem leaves free accuracy on the table.
- Stochastic minibatch → fixed schedule (warmup + cosine/linear decay). Line search on noisy gradients fits the noise of the current batch.
- Diagnosis table: loss diverges immediately → LR above 2/L for the sharpest direction, cut 10×. Loss plateaus early and high → usually too *low* an LR or bad conditioning/scaling, not a local minimum. Loss oscillates with period ~2 steps → at the stability edge, cut 2–3×. Train improves, eval doesn't → not an optimization problem; stop tuning the optimizer.
- Batch size: larger batches cut gradient variance, allowing ~proportionally larger LR up to a critical batch size; beyond it you spend compute for no optimization speedup. Small batches add implicit regularization noise. When you scale batch ×k, start by scaling LR ×k (linear scaling) and re-check stability.

### Convergence diagnostics playbook (symptom → likely cause → first fix)
| Symptom | Likely cause | First fix |
|---|---|---|
| Loss → NaN/inf within a few steps | LR ≫ 2/L, or numerical overflow in the loss | Cut LR 10×; check for exp/log instability separately |
| Loss oscillates with ~2-step period | Sitting at the stability edge of the sharpest direction | Cut LR 2–3×, or add momentum dampening |
| Loss falls then spikes irregularly | Rare large gradients (outlier batches, unclipped) | Global-norm clipping ~1.0; inspect the offending batches |
| Rapid early progress, long flat plateau, gradient ≈ 0 | Saddle/plateau or vanishing signal through a saturating nonlinearity | Better init, LR warm restart, architecture check — not "local minimum" |
| Slow steady progress, gradient norm steady | Ill-conditioning (κ) | Standardize inputs, precondition, adaptive optimizer |
| Train loss falls, validation doesn't | Not an optimization problem | Regularization/data work; stop tuning the optimizer |
| Deterministic solver "converged" instantly | ftol fired on a flat spot, or wrong-scale gtol | Check `result.message`; rescale problem; tighten tolerances |
| Solution violates constraints slightly | Solver tolerance vs your tolerance mismatch | Tighten solver feastol or post-project; never hand-round |

### Constrained problems — the ladder
1. **Eliminate** the constraint by reparameterization: x > 0 → x = exp(z) or softplus(z); simplex → softmax(z); box [l,u] → l + (u−l)·sigmoid(z). Free, exact, and turns the problem unconstrained. Caveat: reparameterization can distort geometry near boundaries (exp compresses gradients as x→0).
2. Simple projectable set (box, ball, simplex) → **projected gradient**; simplex projection is O(n log n) and standard.
3. Convex constraints → model in CVXPY; nonconvex smooth → SLSQP/IPOPT-class interior-point.
4. **Penalties last, and start soft**: a huge penalty from step one wrecks conditioning (κ grows with the weight). Ramp the weight, or use the augmented Lagrangian (multiplier estimate lets a moderate weight enforce the constraint exactly).
5. **Read the multipliers**: λᵢ is the objective's sensitivity to relaxing constraint i (shadow price). λᵢ = 0 → inactive, drop it; huge λᵢ → that constraint is what your objective is paying for — negotiate *it*, not the algorithm.
- KKT in one breath: at a constrained optimum, ∇f is a nonnegative combination of active constraint gradients (stationarity), the point is feasible, multipliers of inequality constraints are ≥ 0, and λᵢgᵢ = 0 (complementary slackness). Checking these four is how you *verify* a constrained answer.

### Reformulation catalog (try these before switching algorithms)
- **Log transform**: products → sums (likelihoods, geometric means); positive variables → unconstrained (x = e^z); posynomials → convex (geometric programming).
- **Slack/epigraph variables**: |·|, max, piecewise-linear → linear constraints; min max f_i → min t with f_i ≤ t.
- **Change of variables to kill κ**: standardize features; optimize log-scale parameters when they span decades (learning rates, regularization weights, chemical concentrations); whiten with a cheap preconditioner.
- **Relax then round**: integrality → LP/convex relaxation (L0 → L1 is the canonical instance); keep the relaxation optimum as a bound on how much rounding cost you.
- **Dualize**: many constraints + few variables → dual has few constraints; decomposable couplings → Lagrangian relaxation splits the problem into independent cheap subproblems coordinated by prices.
- **Eliminate equality constraints by substitution**: Ax = b with A wide → parameterize x = x₀ + Nz (N = null-space basis) and optimize free z.
- **Homogenize/normalize away scale invariance**: if f(cx) = f(x), fix ‖x‖ = 1 rather than letting the optimizer wander along rays (eigenvector-like problems).
- **Decompose by structure**: separable objective + coupling constraint → ADMM/dual decomposition; per-block convexity → alternating minimization (with the caveat that alternating on non-convex couplings can cycle or stall at non-critical points).

### NP-hard recognition reflex
When a subproblem smells like subset selection under interacting constraints, assignment with conflicts, routing, coloring, or "best k features/items with interactions": stop hunting for a poly-time exact trick. Choose deliberately:
(a) **MIP with a time limit** — take the incumbent and report the proven gap;
(b) **convex relaxation** (L1 for L0/cardinality; spectral/SDP for combinatorial) then round, keeping the relaxation's bound;
(c) **greedy with a guarantee** — if the objective is monotone submodular (coverage, diminishing returns), greedy is within (1−1/e) ≈ 0.63 of optimal, usually the right answer to "select k items";
(d) domain heuristic + evaluation harness.
Claiming "here's an efficient exact algorithm" for an NP-hard structure is a critical failure; so is the reverse error of calling a poly-time problem hard (assignment, shortest path, max-flow, isotonic regression, 2-SAT all have exact fast algorithms).

## Failure modes & pitfalls

- **Gradient-descending a problem a solver eats for breakfast.** Assignment (`linear_sum_assignment`), LPs, small QPs, isotonic regression, nonneg least squares (`scipy.optimize.nnls`) have exact solvers. GD gives approximate answers with tuning pain for problems with certified fast solutions.
- **Plain Adam + `weight_decay` believing it regularizes.** It mostly doesn't (coupled decay is normalized away). AdamW or explicit decoupled decay.
- **Penalty weight 1e9 from the start.** Hessian gains eigenvalues of order the weight → κ explodes → GD stalls/oscillates, and you conclude "the constraint makes it hard". Ramp, or augmented Lagrangian.
- **Convexity by eyeball or by wishful algebra.** Products, ratios, and differences of convex functions are generally *not* convex (x/y, x²−x⁴). Verify by DCP composition or a failing CVXPY build. Also don't miss *hidden* convexity: geometric programs (posynomials) become convex under log-log transform; some rational problems become LPs after Charnes–Cooper.
- **Trusting `scipy.optimize.minimize` silently.** Default BFGS with finite-difference gradients on a noisy objective returns garbage without complaint. Always check `result.success`, `result.message`, and the gradient norm at the "solution". If evaluations are noisy, finite differences are meaningless — fix the noise or switch to Nelder-Mead/CMA-ES.
- **One run = the answer, for multimodal problems.** Mixture models, MLE with multiple modes, and most nonconvex fits need multistart: 10–50 random inits, keep the best, *report the spread* — a wide spread is diagnostic information, not an inconvenience.
- **Treating input scaling as preprocessing trivia.** Features spanning 1 vs 10⁶ produce κ ≥ 10¹² on a linear model; no optimizer setting rescues that. Standardize/whiten and the "hard optimization" evaporates. First question for slow convergence: what's the scale spread across inputs and parameters?
- **Confusing "loss stopped improving" with convergence.** Check the gradient norm. Tiny gradient + mediocre loss = plateau/saddle → momentum, LR bump, better init. Large oscillating gradient = LR too high or batch too small. Improving train + flat eval = generalization, not optimization.
- **Optimizer decisions made on the test set.** LR schedules, early stopping, architecture picks tuned on test data quietly become training. Keep a validation split for every optimizer choice.
- **Duality misuse.** Strong duality needs convexity + a constraint qualification (Slater: a strictly feasible point). For nonconvex problems the dual gives a *bound* — valuable for branch-and-bound and Lagrangian relaxation — but reporting the dual optimum as the solution is wrong.
- **Misreading MIP timeouts as failure.** A MIP at its time limit returns the best incumbent *plus a bound*: "within 2.3% of optimal, proven" is a strong result. Set `mip_rel_gap` deliberately instead of burning hours on the last 0.1%.
- **Stopping-criterion sloppiness.** `ftol`-only stopping halts on plateaus that aren't optima; `gtol`-only stalls forever on flat valleys. Use both, plus a max-iteration budget, and report *which* criterion fired.
- **Projected gradient with the wrong projection.** Projecting onto the simplex by "clip negatives, renormalize" is not the Euclidean projection and biases the solution; use the exact sorting-based projection or a softmax reparameterization.
- **Comparing optimizers at one LR each.** Any A-vs-B optimizer claim requires tuning LR (at least a small grid) *per optimizer*; Adam-at-its-best vs SGD-at-default is the most common source of folklore.
- **Alternating minimization treated as guaranteed.** Coordinate/alternating schemes monotonically decrease the objective but can converge to points that are not even stationary for the joint problem when blocks couple non-smoothly. Verify the *joint* gradient/KKT conditions at the end, not just per-block optimality.
- **Ignoring integrality until the end, then rounding a fractional LP solution by hand.** Naive rounding can be infeasible or arbitrarily bad for covering/packing structures. Either use the MIP directly (branch-and-bound does principled rounding) or use a rounding scheme with a guarantee (randomized rounding, iterative rounding) and check feasibility explicitly.
- **Stochastic gradient estimates with unacknowledged bias.** Minibatching is unbiased for sums; it is *not* unbiased through nonlinearities (log of a minibatch mean, ratios of minibatch sums, clipped losses). Biased gradients converge to the wrong point while all diagnostics look healthy — check whether your estimator sits inside or outside the nonlinearity.

## Worked micro-examples

**1. Reformulation: L1 regression as an LP (no subgradient hacks).**
min ‖Ax − b‖₁ becomes: introduce t ∈ ℝᵐ, minimize Σtᵢ subject to −t ≤ Ax − b ≤ t.
```python
import cvxpy as cp
x = cp.Variable(n)
prob = cp.Problem(cp.Minimize(cp.norm1(A @ x - b)))
prob.solve()          # CVXPY performs the LP reformulation internally — exact, fast
```
The same slack-variable trick converts minimax (min max_i fᵢ becomes min t s.t. fᵢ ≤ t) and any |·| into linear form. Recognizing "this nonsmooth thing is secretly an LP" is worth more than any subgradient schedule.

**2. Condition number → concrete step-count arithmetic.**
f(x) = ½(x₁² + 100x₂²): L = 100, μ = 1, κ = 100. Best fixed step η* = 2/(L+μ) ≈ 0.0198; contraction (κ−1)/(κ+1) ≈ 0.98 per step → ~115 steps per e-fold, ~1150 steps for 10⁻⁴ error. Heavy-ball momentum: (√κ−1)/(√κ+1) ≈ 0.82 → ~12 steps per e-fold, ~10× fewer. Rescale x₂' = 10x₂ and κ = 1: converges in one well-stepped iteration. Same problem, three costs — conditioning is the variable, and "rescale" beat both algorithms.

**3. KKT reading on a budget allocation.**
max Σᵢ αᵢ log xᵢ s.t. Σxᵢ = B, xᵢ ≥ 0. Stationarity: αᵢ/xᵢ = λ → xᵢ = αᵢ/λ; the budget constraint gives λ = Σαⱼ/B, so xᵢ = B·αᵢ/Σαⱼ — proportional allocation, derived rather than guessed. The multiplier λ = Σαⱼ/B is the marginal utility of budget: doubling B halves λ, quantifying exactly what another unit of budget buys. Log-utility + linear budget → proportional split is a pattern worth caching; it's why softmax-shaped allocations keep reappearing.

**4. Submodular selection with a guarantee.**
"Pick 10 monitoring locations covering the most incidents" — coverage is monotone submodular, so greedy (repeatedly add the location covering the most *uncovered* incidents) is guaranteed ≥ (1−1/e) ≈ 63% of the optimal coverage, and in practice lands within a few percent. The alternative — an exact search — is NP-hard set cover. Greedy here isn't a compromise; for this objective class it's provably near the best any polynomial algorithm can do. Recognize the diminishing-returns property, cite the guarantee, ship the greedy.

## Verification & self-check

- **Check certificates, not just objective values**: gradient norm ≈ 0 (unconstrained); KKT residuals + complementary slackness (constrained); duality gap (convex solvers report it); MIP gap (branch-and-bound reports it).
- **Perturb the "optimum"**: evaluate f at x* ± small random steps; anything lower means you weren't done. Catches premature-stop and wrong-sign bugs cheaply.
- **Verify custom gradients** against central finite differences at random points (`scipy.optimize.check_grad`); relative error ≲ 1e-6 in fp64. Most "the optimizer is broken" reports are wrong gradients.
- **Feasibility first**: plug the answer into every constraint and report the max violation with its tolerance; solver "optimal" can hide 1e-4 violations that matter downstream.
- **Cross-check with a second method on a shrunk instance**: brute force at n = 10 vs your method; solver A vs solver B. Objective disagreement beyond tolerance = modeling bug, usually a sign flip or a missing constraint.
- **Multistart audit for nonconvex claims**: an asserted optimum should come with the restart spread that supports it.
- **Sanity-check the multipliers**: shadow prices with impossible signs (negative price for a resource you'd pay for) indicate a modeling error even when the solver reports success.
