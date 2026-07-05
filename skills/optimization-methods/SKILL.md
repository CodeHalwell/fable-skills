---
name: optimization-methods
description: Loads when solving or debugging optimization problems — choosing between gradient methods, LP/QP/MIP solvers, and heuristics; diagnosing slow/diverging training; applying Lagrangians/KKT; tuning Adam vs SGD; recognizing convexity or NP-hard structure; or reformulating a problem (relaxation, change of variables, dual) to make it tractable.
---

# Optimization Across Math and ML

Most of this domain is strong-model baseline (L1→LP, AdamW vs Adam, κ arithmetic, submodular greedy, Slater, MIP gaps, simplex projection). This sheet keeps the routing tables, the reformulation catalog, and the corrections that matter.

## First move: reformulate, not tune

An hour of reformulation routinely beats a week of solver tuning. Catalog (one-liners; run through it before switching algorithms):
- Log transform: products → sums; x>0 → x=eᶻ; posynomials → convex (geometric programming).
- Slack/epigraph: |·|, max, minimax → linear constraints (min max fᵢ → min t s.t. fᵢ ≤ t).
- Change variables to kill κ: standardize features; optimize log-scale for parameters spanning decades.
- Relax then round: L0 → L1 is the canonical case; keep the relaxation optimum as a bound on rounding cost.
- Dualize: many constraints + few variables → few-constraint dual; decomposable couplings → Lagrangian relaxation into priced subproblems.
- Eliminate equalities by substitution (x = x₀ + Nz over the null space); fix ‖x‖=1 to kill scale invariance; ADMM/alternating for block structure (verify the *joint* KKT at the end — alternating schemes can stall at non-stationary points of the joint problem).
- Convexity is compositional (DCP rules), not eyeball: products/ratios/differences of convex functions are generally not convex; a failing CVXPY build is the practical test. Watch for *hidden* convexity (GP log-log transform, Charnes–Cooper for ratios).

## Solver routing (one-liners)

Smooth deterministic full-gradient → L-BFGS (line search included; hand-tuned GD is strictly worse here). Stochastic minibatch → AdamW/SGD+momentum (L-BFGS and line searches break under gradient noise — they fit the batch's noise). LP → HiGHS via `linprog`; convex QP/conic → OSQP/Clarabel via CVXPY. Integer + guarantees → MIP or CP-SAT. Expensive black-box ≤~10³ evals, ≤~20 dims → Bayesian opt. Cheap rugged black-box ≤~100 dims → CMA-ES. Nonlinear least squares → `least_squares` (LM/TRF), never generic `minimize`. Assignment → `linear_sum_assignment` — exact poly-time, not a search problem.

## Anchors and corrections

- **κ arithmetic (with the right units):** GD contraction (κ−1)/(κ+1); steps per e-fold ≈ κ/2, per decimal digit ≈ 1.15κ. κ=100: ~50 steps/e-fold GD, ~5 with heavy-ball ((√κ−1)/(√κ+1)); Newton/rescaling: 1 step. (Unit slip alert: an earlier revision quoted per-digit counts as "per e-fold".) Rescaling inputs often beats both algorithm upgrades — first question for slow convergence is the scale spread across inputs/parameters, since features at 1 vs 10⁶ give κ ≥ 10¹² that no optimizer setting rescues.
- **Diagnosis table (the counterintuitive rows):** loss plateaus early *and high* → usually LR too **low** or bad scaling, not a local minimum; tiny gradient + mediocre loss → saddle/plateau (in high dimensions bad critical points are overwhelmingly saddles; "stuck in a local minimum" is usually the wrong diagnosis); ~2-step oscillation → at the 2/L stability edge, cut LR 2–3×; falls-then-spikes → unclipped rare large gradients, clip global norm ~1.0; train improves but eval doesn't → not an optimization problem, stop tuning the optimizer; deterministic solver "converged" instantly → ftol fired on a flat spot — read `result.message` and report *which* criterion fired.
- **Adam specifics:** `torch.optim.Adam(weight_decay=...)` is coupled L2 that the second-moment division largely neutralizes — use AdamW; this dissolved most "Adam generalizes worse" folklore. ε is a real hyperparameter (raise 1e-8 → 1e-4+ in low-signal/RL regimes to damp spikes). Warmup exists because early v̂ is high-variance, mis-scaling the first steps. Batch ×k → start with LR ×k and re-check stability; beyond the critical batch size you buy nothing.
- **Constraints ladder:** (1) reparameterize away (exp/softplus, softmax, sigmoid-box) — but transforms saturate near boundaries, so if the optimum sits *on* the boundary prefer (2) projection (box = clip; simplex = the sort-and-threshold **shift**, never "clip negatives and renormalize" — projection translates, renormalizing rescales, different point); (3) CVXPY/IPOPT; (4) penalties *last and soft* — weight 1e9 from step one puts a 1e9 eigenvalue in the Hessian and stalls everything; ramp, or use the augmented Lagrangian.
- **Read the multipliers:** λᵢ = shadow price of constraint i. λ=0 → drop it; huge λ → negotiate that constraint, not the algorithm. Impossible signs (negative price for a resource you'd pay for) = modeling bug even when the solver reports success.
- **Duality:** strong duality = convexity + Slater (strictly feasible point); LPs need only feasibility. Nonconvex dual = a bound, never the solution — and the dual maximizer's primal reconstruction may be infeasible.
- **NP-hard reflex:** subset selection under interacting constraints / routing / coloring → (a) MIP with a time limit — the incumbent *plus proven gap* is a strong result, set `mip_rel_gap` deliberately; (b) relaxation + rounding with the bound kept; (c) monotone submodular + cardinality → greedy is (1−1/e) ≈ 0.63-optimal and (Feige) best-possible poly-time; (d) heuristic + evaluation harness. Equal-and-opposite failure: calling assignment, max-flow, 2-SAT, isotonic regression, or shortest path "intractable".
- **Stochastic-estimator bias:** minibatching is unbiased for sums, *not* through nonlinearities (log of batch mean, ratios of batch sums, in-batch normalizers/contrastive denominators, clipped losses). Biased gradients converge to the wrong point while every diagnostic looks healthy — check whether the estimator sits inside or outside the nonlinearity.
- **Objective/metric mismatch:** if the surrogate (cross-entropy) no longer correlates with the judged metric (F1, top-k, revenue), the optimizer can succeed while the metric stalls — change the surrogate or tune the threshold; don't touch optimizer knobs. Kinks where the optimum lives (abs, max, quantiles, sorting) stall subgradient methods — smooth them (huber, log-sum-exp, soft-sort).
- **Noisy objectives:** `scipy.optimize.minimize` defaults (BFGS + finite differences) return garbage without complaint — check `result.success`/`message`/gradient norm always; on noise switch to CMA-ES/Nelder-Mead/model-based DFO or fix the noise. Multimodal fits need multistart (10–50 inits) and the *spread reported* — a wide spread is diagnostic information.
- **Comparing optimizers at one LR each is folklore manufacturing** — tune LR per optimizer (log-scale grid; random search beats grid for >2 hyperparameters). Optimizer decisions tuned on the test set are training on it.

## Verification & self-check

- Certificates, not vibes: gradient norm (unconstrained), KKT residuals + complementary slackness (constrained), duality gap (convex), MIP gap (integer). An answer without its certificate is a guess.
- Perturb the "optimum" (x* ± small random steps); anything lower means not done.
- `check_grad` custom gradients against central finite differences (rel err ≲ 1e-6 fp64) — most "broken optimizer" reports are wrong gradients.
- Feasibility first: plug the answer into every constraint, report max violation vs tolerance; "optimal" can hide 1e-4 violations that matter downstream. Re-evaluate the objective from the returned variables independently of the solver's reported value — catches units/sign/forgotten-term modeling bugs.
- Cross-check on a shrunk instance (brute force at n=10, or solver A vs B). Scale test: multiply all inputs by 10 and confirm the solution transforms as the math says; if not, a hard-coded tolerance is acting as a hidden constraint.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened: saddle-vs-local-minimum diagnosis and the plateau→LR-too-low row), 0 delta.
- Opus 4.8 nailed L1→LP, AdamW mechanics, κ/momentum arithmetic, penalty-weight stiffness, (1−1/e) with the Feige optimality bound, simplex projection (shift not rescale), Slater, MIP-gap reporting, minibatch-bias examples, tool routing.
- Audit's main find was internal: step-count figures labeled "per e-fold" were per decimal digit (~2.3× off); corrected here. Retained value: reformulation catalog completeness, diagnosis table, verification certificates.
