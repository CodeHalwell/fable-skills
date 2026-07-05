---
name: simulation-and-scientific-computing
description: Load when building or debugging numerical simulations and scientific computation — ODE/PDE solving, Monte Carlo methods, stochastic models, scipy/NumPy/JAX numerical code, solver selection and tolerances, reproducibility with random seeds, unit handling, or validating simulation correctness and performance.
---

# Simulation & Scientific Computing

## Core mental model

1. **Climb the modeling ladder from the bottom.** Analytic/closed-form solution > deterministic ODE integration > PDE/spatial model > agent-based/Monte Carlo. Each rung costs 10–1000× more compute and debugging than the one below. Always ask "does the cheaper rung answer the actual question?" before building. The rungs break upward for identifiable reasons: analytic breaks on nonlinearity/heterogeneity; mean-field ODEs break when discreteness, spatial structure, or fluctuations matter (small populations, extinction events, network effects); deterministic breaks when the *distribution* of outcomes is the question, not the mean.
2. **A simulation result without validation is a random number with good typography.** Plausible-looking output is the default failure mode — the code runs, the plot is smooth, and it's wrong by 2× because of a sign, a unit, or an unconverged tolerance. Validation (conservation laws, limiting cases, convergence studies) is not a final step; it's the definition of done.
3. **Numerical error is a budget you set, not a fact you receive.** Solver tolerances, step sizes, grid resolutions, and Monte Carlo N are all knobs trading cost for error. The expert states the target accuracy first ("answers good to 1%"), then tunes knobs to it — never runs at defaults and reports whatever comes out (default-blindness).
4. **Reproducibility and statistical independence are both seed problems.** Every stochastic run must be exactly rerunnable (explicit seeds, logged with results) AND replicated runs must be statistically independent (distinct, non-overlapping streams — not `seed=42` everywhere, not `seed=i` folklore but a proper spawning mechanism).
5. **Vectorize before you parallelize; both before you rewrite.** The Python interpreted-loop tax is ~100× versus vectorized NumPy. Most "we need a cluster/C++" conversations end with a vectorization pass and a 50× speedup on one core.
6. **Units are a correctness class, not a formatting nicety.** Mars Climate Orbiter–class failures come from raw floats with implicit units. Enforce a convention (SI everywhere, convert at the boundary) or a library (`pint`; unit-suffixed variable names as the minimum: `dt_s`, `k_per_hour`).

## Solver selection: ODEs and the stiffness question

Reasoning chain for any ODE system:
1. **Is it stiff?** Stiffness = timescales spanning orders of magnitude (fast transient + slow evolution: chemical kinetics, circuits, reaction networks are stiff by default). Detection without theory: **the failed-explicit-solver signature** — an explicit method (RK45) grinds to millions of tiny steps or blows up even though the solution looks smooth and boring. If RK45 is slow on a smooth solution, it's stiff; stop tuning tolerances and switch method families.
2. **scipy method choice (`scipy.integrate.solve_ivp`):** `RK45` default for non-stiff; `Radau` or `BDF` for stiff; `LSODA` when unsure (auto-switches, good triage tool — if LSODA is much faster than RK45, that *is* your stiffness diagnosis). Supply the Jacobian (or its sparsity pattern) for stiff systems of dimension ≳ dozens — the implicit solvers otherwise burn their time finite-differencing it.
3. **Tolerances are the accuracy contract:** `rtol` = relative error per step, `atol` = absolute floor *per component* — and `atol` must scale with each state variable's magnitude. The classic bug: concentrations ranging 1e-12 to 1e-3 with default `atol=1e-6` means the small species are pure noise. Set `atol ≈ 1e-6 × typical_scale(component)` as a vector. Then verify: tighten both by 10× and confirm the answer moves less than your error budget — if it moves, you were reporting solver artifact, not solution.
4. **Events, not step-and-check:** use `events=` (root-finding on a continuous function) for threshold crossings; checking `y > threshold` at solver-chosen output points misses crossings between points.
5. **PDEs:** method-of-lines (discretize space, hand the stiff ODE system to BDF/Radau) is the pragmatic default for parabolic problems; diffusion terms make the semi-discrete system stiff — explicit time-stepping is bound by the CFL-type constraint `dt ∝ dx²`, which is the standard "why does halving dx make it 8× slower or unstable" answer. Reach for real frameworks (FEniCS/Firedrake/Dedalus-class) before hand-rolling 2D+ solvers.

## Monte Carlo engineering

- **The 1/√N economics govern everything:** standard error ∝ σ/√N, so one more digit of accuracy costs 100× the compute. Consequences: (a) always report MC results as estimate ± standard error (an MC number without an error bar is meaningless); (b) when N×100 is unaffordable, the answer is *variance reduction*, not patience.
- **Variance reduction, applied in order of cheapness:** *antithetic variates* (pair U with 1−U; near-free, helps monotone integrands), *control variates* (subtract a correlated quantity with known expectation — e.g., price the exotic option minus β×(geometric-Asian analytic price); variance drops by 1−ρ²), *importance sampling* (mandatory for rare events: sampling a 1e-6-probability tail directly needs ~1e8 draws for 10% error — instead sample from a shifted distribution and reweight by the likelihood ratio; the diagnostic for a bad proposal is effective sample size collapse / a few weights dominating), *quasi-Monte Carlo* (Sobol sequences via `scipy.stats.qmc`, near-1/N convergence for smooth moderate-dimension integrands — use scrambled Sobol to retain error estimates).
- **Seed management done right (NumPy, as of 2026):** use the `Generator` API, never the legacy global `np.random.seed()`. For parallel/replicated runs, spawn independent streams: `SeedSequence(base_seed).spawn(n_workers)` → one `default_rng(child)` per worker. This guarantees both reproducibility (base seed logged) and independence (streams provably non-overlapping). Worker ID arithmetic on seeds (`seed + rank`) is the folklore anti-pattern that can correlate streams.
- **Convergence check that actually works:** run at N and 4N; the estimate should move by about the standard error and the reported error bar should halve. If the estimate jumps by many standard errors, something is biased or a rare regime is undersampled — more N will not save you.

## JAX-era scientific computing (as of 2026)

The ecosystem is mature: **JAX** (jit/vmap/grad as the primitives), **diffrax** (autodifferentiable ODE/SDE solvers), **equinox** (models as PyTrees), **NumPyro** (the dominant JAX probabilistic programming library), **optax**/**BlackJAX** alongside. When JAX beats NumPy: (1) you need gradients through the simulation (calibration, sensitivity analysis, neural ODEs — this is the killer feature: `grad` through a diffrax solve replaces finite-difference parameter sweeps), (2) GPU/TPU-friendly workloads, (3) embarrassing batch parallelism expressed as `vmap` (10⁴ parameter sets integrated at once with one decorator). When NumPy/SciPy wins: one-off CPU computation, heavy Python-side control flow, tiny problems (jit compile time exceeds runtime), teams unfamiliar with functional constraints. The disciplines JAX enforces: **pure functions** (no side effects, no in-place mutation — `x.at[i].set(v)` instead of `x[i] = v`), explicit PRNG keys (`jax.random.split(key)` — a *feature* for the seed-independence problem above), Python control flow on traced values must become `lax.cond`/`lax.scan`, and float32 is the default (set `jax.config.update("jax_enable_x64", True)` for scientific work or chase phantom precision bugs — this is the #1 JAX-for-science gotcha).

## Performance reasoning

Do the arithmetic before optimizing: a Python loop executes ~10⁷ simple iterations/second; vectorized NumPy does ~10⁹ element-ops/second. So a 10⁸-element triple-nested loop is ~hours in Python, ~seconds vectorized — *that* is why "vectorize first" precedes any parallelism talk (parallelizing interpreted Python buys at most core-count× against a 100× interpreter tax, and `multiprocessing` adds pickling overhead that often eats it). Order of operations: profile (find the actual hot loop — it's rarely where you think), vectorize it (NumPy broadcasting / `np.einsum`), then `numba.njit` or JAX `jit` for genuinely sequential loops that can't vectorize (time-stepping recurrences), then parallelize (vmap/multiprocessing), then and only then consider a compiled-language rewrite. Memory bandwidth is the ceiling vectorization hits: chains of large temporaries (`a*b + c*d` on GB arrays) thrash cache — fuse with numba/JAX jit when that becomes the bottleneck.

## Stochastic vs deterministic judgment

Use a deterministic (ODE/mean-field) model when populations are large, the question is about means/trajectories, and noise averages out. Go stochastic when: counts are small enough that discreteness matters (extinction: an ODE predicts 0.3 infected individuals forever; reality goes extinct), the question is about distributions/percentiles/rare events, or fluctuations feed back nonlinearly. The hybrid prior: run the deterministic model first *anyway* — it's the debugging baseline and the control variate; the stochastic model's mean should approach the ODE in the large-N limit, and checking that IS a validation test (Gillespie/SSA mean vs mass-action ODE is the canonical pair).

## How an expert thinks through it: "chemical kinetics sim takes 6 hours and the results look noisy"

Internal monologue: *Two complaints — slow and noisy — and my prior says they share a cause: an explicit solver fighting stiffness.* Look at the code: `solve_ivp(f, tspan, y0)` — default RK45, default tolerances, and the step-count is astronomical for a smooth solution. Kinetics with rate constants spanning 1e-3 to 1e6: stiff, textbook. *The "noise" hypothesis check: is it real stochasticity? No — the model is deterministic; the wiggle is the solver bouncing off its stability boundary.* Switch to `method='BDF'`, supply the Jacobian (kinetics Jacobians are cheap analytically). Runtime: 6 hours → 40 seconds. *Not done — considered stopping here, rejected: defaults are still lying.* Species concentrations span 1e-12 to 1e-2; default `atol=1e-6` makes trace species garbage. Set vector atol per species; tighten rtol 1e-3→1e-6 and compare: dominant species stable to 5 digits, trace species change by 30% — so the earlier trace results *were* artifact. *Considered rewriting in Julia/JAX for speed — rejected: 40s is inside the budget; that's the stopping rule.* Final validation: mass conservation — total atoms drift < 1e-9 over the run (BDF isn't conservative by construction, so drift is a real diagnostic); equilibrium constants from the long-time limit match the input thermodynamics; halving rtol changes reported quantities by less than the 1% error budget. Now it's a result.

## Failure modes & pitfalls

- **Default-blindness:** reporting `solve_ivp` output at `rtol=1e-3` (the default — three digits, per step, at best) for a result claimed to 4 digits; MC without error bars; a grid resolution chosen by memory rather than convergence. Every reported digit must be backed by a knob-tightening test.
- **Scalar `atol` across mixed-magnitude state** (above) — silently zeros-out small components.
- **The failed-explicit-solver misdiagnosis:** responding to RK45 slowness by loosening tolerances (get garbage faster) or shrinking `max_step` (get slower) instead of switching to BDF/Radau.
- **Accumulating time as `t += dt` in float32** or comparing floats with `==` for loop termination — drift and off-by-one-step; compute `t = i*dt` and compare with tolerance.
- **`np.random.seed(42)` in library code / same seed in every worker:** N "independent" replicas of the same stream. Use `SeedSequence.spawn`; in JAX, split keys — never reuse one.
- **Unit mixing:** a rate in 1/hour meets a dt in seconds; nothing errors, everything is wrong by 3600. Either `pint` at the interfaces or a single-unit-system rule enforced in review; check the final answer's plausibility dimensionally (e.g., a diffusion coefficient's magnitude in the units claimed).
- **Validating only the happy path:** conservation laws are *free* tests (mass, energy, probability sums to 1, symplectic invariants) — assert them in code, not by eyeball. Limiting cases (rate→0 recovers the simpler model; N→∞ recovers the ODE) catch structural bugs that convergence studies miss.
- **Convergence study done wrong:** halving the step and seeing "the answer barely changed" is necessary but weak — check the *order*: for a method of order p, halving h should shrink the error against a reference (finest-grid or analytic) by ~2^p. Wrong observed order = a bug in the discretization or a non-smooth solution violating the method's assumptions (this test catches sign errors that "looks converged" never will).
- **jit-compiling code with Python-side branching on traced values** (JAX `TracerBoolConversionError`) or benchmarking JAX without `.block_until_ready()` (async dispatch makes everything look instant).
- **Chasing parallelism before vectorization** (the arithmetic above), and `multiprocessing` over arrays without shared memory — pickling costs dominate.

## Worked micro-example: stiff solve + tolerance discipline (scipy)

```python
import numpy as np
from scipy.integrate import solve_ivp

# Robertson problem — the canonical stiff kinetics test (rates span 9 orders).
def f(t, y):
    return np.array([-0.04*y[0] + 1e4*y[1]*y[2],
                      0.04*y[0] - 1e4*y[1]*y[2] - 3e7*y[1]**2,
                      3e7*y[1]**2])
def jac(t, y):
    return np.array([[-0.04,  1e4*y[2],           1e4*y[1]],
                     [ 0.04, -1e4*y[2] - 6e7*y[1], -1e4*y[1]],
                     [ 0.0,   6e7*y[1],            0.0]])

y0 = [1.0, 0.0, 0.0]
atol = np.array([1e-8, 1e-12, 1e-8])   # y[1] lives near 1e-5 max: per-component floor
sol = solve_ivp(f, (0, 1e5), y0, method="BDF", jac=jac, rtol=1e-6, atol=atol,
                dense_output=True)
assert sol.success
# Free validation: y is a probability-like triple — conservation must hold.
drift = np.abs(sol.y.sum(axis=0) - 1.0).max()
assert drift < 1e-7, f"conservation violated: {drift:.2e}"
# Tolerance check: re-run with rtol=1e-8; reported quantities must move < error budget.
# (RK45 on this problem: ~minutes and millions of steps — the stiffness signature.)
```

## Verification / self-check

Before presenting any numerical result, confirm all of:
- **Conservation/invariants** asserted in code and passing (pick whatever the physics gives you for free).
- **Limiting case** reproduced (parameter → 0/∞ recovers a known simpler answer, or analytic special case matched to expected digits).
- **Convergence with order check**: tighten tolerance / halve step / double N — the answer moves less than the stated error budget, and the error shrinks at the method's theoretical rate.
- **Reproducibility**: seeds/versions logged; rerun produces identical (deterministic) or statistically consistent (stochastic, within stated error bars) output.
- **Units audit** on inputs and the headline number.
- Stopping rules: stop refining when numerical error is comfortably below *model* error (parameter uncertainty usually dominates — a 1%-accurate solve of a model with 20% parameter error is done); stop optimizing when runtime fits the iteration loop you actually need (a 40-second sim run 100 times/day needs no rewrite); stop climbing the modeling ladder when the current rung answers the question asked, with error bars.
