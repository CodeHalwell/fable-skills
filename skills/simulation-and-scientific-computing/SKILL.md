---
name: simulation-and-scientific-computing
description: Load when building or debugging numerical simulations and scientific computation — ODE/PDE solving, Monte Carlo methods, stochastic models, scipy/NumPy/JAX numerical code, solver selection and tolerances, reproducibility with random seeds, unit handling, or validating simulation correctness and performance.
---

# Simulation & Scientific Computing

## Core mental model

1. **Climb the modeling ladder from the bottom.** Analytic/closed-form solution > deterministic
   ODE integration > PDE/spatial model > agent-based/Monte Carlo. Each rung costs 10–1000× more
   compute and debugging than the one below. Always ask "does the cheaper rung answer the actual
   question?" before building. The rungs break upward for identifiable reasons:
   - analytic breaks on nonlinearity and heterogeneity;
   - mean-field ODEs break when discreteness, spatial structure, or fluctuations matter (small
     populations, extinction events, network effects);
   - deterministic breaks when the *distribution* of outcomes is the question, not the mean.
2. **A simulation result without validation is a random number with good typography.**
   Plausible-looking output is the default failure mode — the code runs, the plot is smooth, and
   it's wrong by 2× from a sign, a unit, or an unconverged tolerance. Validation is not a final
   step; it's the definition of done.
3. **Numerical error is a budget you set, not a fact you receive.** Tolerances, step sizes, grid
   resolutions, and Monte Carlo N all trade cost for error. State the target accuracy first
   ("answers good to 1%"), then tune knobs to it. Running at defaults and reporting whatever comes
   out is default-blindness.
4. **Reproducibility and statistical independence are both seed problems.** Every stochastic run
   must be exactly rerunnable (explicit seeds, logged with results) AND replicated runs must be
   statistically independent (distinct, provably non-overlapping streams — not `seed=42`
   everywhere, and not `seed+rank` folklore).
5. **Vectorize before you parallelize; both before you rewrite.** The Python interpreted-loop tax
   is roughly 100× versus vectorized NumPy. Most "we need a cluster / C++" conversations end with
   a vectorization pass and a 50× speedup on one core.
6. **Units are a correctness class, not a formatting nicety.** Mars-Climate-Orbiter-class failures
   come from raw floats with implicit units. Enforce a convention (SI everywhere, convert at the
   boundary), a library (`pint`), or at minimum unit-suffixed names (`dt_s`, `k_per_hour`).

## Solver selection: ODEs and the stiffness question

Reasoning chain for any ODE system:

1. **Is it stiff?** Stiffness = timescales spanning orders of magnitude (fast transient + slow
   evolution). Chemical kinetics, circuits, and reaction networks are stiff by default. Detection
   without theory — **the failed-explicit-solver signature**: an explicit method (RK45) grinds
   through millions of tiny steps or blows up even though the solution looks smooth and boring.
   If RK45 is slow on a smooth solution, it's stiff. Stop tuning tolerances; switch method family.
2. **scipy method choice** (`scipy.integrate.solve_ivp`):
   - `RK45` — default for non-stiff problems.
   - `Radau` or `BDF` — stiff problems.
   - `LSODA` — auto-switching; the triage tool when unsure. If LSODA is much faster than RK45,
     that *is* your stiffness diagnosis.
   - Supply the Jacobian (or its sparsity pattern) for stiff systems beyond a few dozen
     dimensions — implicit solvers otherwise burn their time finite-differencing it.
3. **Tolerances are the accuracy contract.** `rtol` = relative error per step; `atol` = absolute
   floor *per component* — and `atol` must scale with each state variable's magnitude. The classic
   bug: concentrations spanning 1e-12 to 1e-3 with the default `atol=1e-6` means the small species
   are pure noise. Pass a vector: `atol ≈ 1e-6 × typical_scale(component)`. Then verify: tighten
   both by 10× and confirm the answer moves less than your error budget — if it moves more, you
   were reporting solver artifact, not solution.
4. **Events, not step-and-check.** Use `events=` (root-finding on a continuous event function) for
   threshold crossings; checking `y > threshold` at solver output points misses crossings between
   points.
5. **PDEs:** method-of-lines — discretize space, hand the resulting (stiff) ODE system to
   BDF/Radau — is the pragmatic default for parabolic problems. Diffusion terms make the
   semi-discrete system stiff; explicit time-stepping is bound by the CFL-type constraint
   `dt ∝ dx²`, which is the standard answer to "why did halving dx make it 8× slower / unstable".
   Reach for real frameworks (FEniCS/Firedrake/Dedalus-class) before hand-rolling 2D+ solvers.

## Monte Carlo engineering

- **The 1/√N economics govern everything.** Standard error ∝ σ/√N: one more digit of accuracy
  costs 100× the compute. Consequences:
  - Always report MC results as estimate ± standard error. An MC number without an error bar is
    meaningless.
  - When 100× N is unaffordable, the answer is variance reduction, not patience.
- **Variance reduction, applied in order of cheapness:**
  - *Antithetic variates*: pair U with 1−U. Near-free; helps monotone integrands.
  - *Control variates*: subtract a correlated quantity with known expectation — e.g., price the
    exotic option minus β × (geometric-Asian analytic price); variance drops by the factor 1−ρ².
  - *Importance sampling*: mandatory for rare events. Sampling a 1e-6-probability tail directly
    needs ~1e8 draws for 10% relative error; instead sample a shifted distribution and reweight
    by the likelihood ratio. Diagnostic of a bad proposal: effective sample size collapses — a
    few weights dominate the sum.
  - *Quasi-Monte Carlo*: scrambled Sobol sequences (`scipy.stats.qmc.Sobol`) give near-1/N
    convergence for smooth, moderate-dimension integrands, and scrambling retains error estimates.
- **Seed management done right (NumPy, as of 2026):** use the `Generator` API; never the legacy
  global `np.random.seed()`. For parallel or replicated runs, spawn independent streams:
  ```python
  from numpy.random import SeedSequence, default_rng
  children = SeedSequence(base_seed).spawn(n_workers)
  rngs = [default_rng(c) for c in children]   # reproducible AND independent
  ```
  Worker-ID arithmetic on seeds (`seed + rank`) is the folklore anti-pattern that can produce
  correlated streams.
- **A convergence check that actually works:** run at N and at 4N. The estimate should move by
  about one standard error, and the reported error bar should halve. If the estimate jumps by many
  standard errors, something is biased or a rare regime is undersampled — more N will not save you.

## JAX-era scientific computing (as of 2026)

The ecosystem is mature: **JAX** (with `jit`/`vmap`/`grad` as the new primitives), **diffrax**
(autodifferentiable ODE/SDE solvers), **equinox** (models as PyTrees), **NumPyro** (the dominant
JAX probabilistic-programming library), with **optax** and **BlackJAX** alongside.

When JAX beats NumPy/SciPy:
- You need **gradients through the simulation** — calibration, sensitivity analysis, neural ODEs.
  This is the killer feature: `grad` through a diffrax solve replaces finite-difference parameter
  sweeps entirely.
- GPU/TPU-friendly array workloads.
- Embarrassing batch parallelism expressed as `vmap`: 10⁴ parameter sets integrated at once with
  one decorator.

When NumPy/SciPy wins: one-off CPU computation, heavy Python-side control flow, tiny problems
(jit compile time exceeds runtime), teams unfamiliar with the functional constraints.

The disciplines JAX enforces:
- **Pure functions**: no side effects, no in-place mutation — `x.at[i].set(v)`, not `x[i] = v`.
- **Explicit PRNG keys**: `jax.random.split(key)` — never reuse a key. This is a *feature* for
  the seed-independence problem above.
- Python control flow on traced values must become `lax.cond` / `lax.scan` / `lax.while_loop`.
- **Float32 is the default.** For scientific work set
  `jax.config.update("jax_enable_x64", True)` or spend a week chasing phantom precision bugs —
  the #1 JAX-for-science gotcha.

## Performance reasoning

Do the arithmetic before optimizing:
- A Python loop runs ~10⁷ simple iterations/second; vectorized NumPy does ~10⁹
  element-operations/second. A 10⁸-element nested loop is hours in Python, seconds vectorized.
- That is why "vectorize first" precedes any parallelism talk: parallelizing interpreted Python
  buys at most core-count× against a 100× interpreter tax, and `multiprocessing` adds pickling
  overhead that often eats the gain on array workloads.

Order of operations: **profile** (find the actual hot loop — it's rarely where you think) →
**vectorize** (broadcasting, `np.einsum`) → **jit** (`numba.njit` or JAX `jit`) for genuinely
sequential recurrences that can't vectorize (time-stepping loops) → **parallelize**
(`vmap`, multiprocessing with shared memory) → only then consider a compiled-language rewrite.
Memory bandwidth is the ceiling vectorization hits: chains of large temporaries (`a*b + c*d` on
GB-scale arrays) thrash cache — fuse them with numba/JAX jit when that becomes the bottleneck.

## Stochastic vs deterministic model judgment

- Choose deterministic (ODE / mean-field) when populations are large, the question is about means
  and trajectories, and noise averages out.
- Go stochastic when: counts are small enough that discreteness matters (an ODE predicts 0.3
  infected individuals persisting forever; reality goes extinct), the question is about
  distributions/percentiles/rare events, or fluctuations feed back nonlinearly.
- The hybrid prior: build the deterministic model first *anyway* — it's the debugging baseline
  and a natural control variate, and the stochastic model's mean must approach it in the large-N
  limit. Checking that limit IS a validation test (Gillespie/SSA mean vs the mass-action ODE is
  the canonical pair).

## How an expert thinks through it: "kinetics sim takes 6 hours and looks noisy"

*Two complaints — slow and noisy — and my prior says they share a cause: an explicit solver
fighting stiffness.* The code: `solve_ivp(f, tspan, y0)` — default RK45, default tolerances — and
the step count is astronomical for a smooth solution. Rate constants span 1e-3 to 1e6: stiff,
textbook.

*Check the "noise" hypothesis: is it real stochasticity? No — the model is deterministic; the
wiggle is the solver bouncing along its stability boundary.*

Switch to `method='BDF'` and supply the Jacobian (kinetics Jacobians are cheap analytically).
Runtime: 6 hours → 40 seconds.

*Considered stopping here — rejected: the defaults are still lying.* Species concentrations span
1e-12 to 1e-2; the default scalar `atol=1e-6` makes trace species garbage. Set a per-species atol
vector; tighten `rtol` from 1e-3 to 1e-6 and compare: dominant species stable to 5 digits, trace
species move 30% — so the earlier trace results *were* artifact.

*Considered rewriting in JAX/Julia for speed — rejected: 40 seconds is inside the iteration
budget; that's the stopping rule.*

Final validation: mass conservation drifts < 1e-9 over the run (BDF isn't conservative by
construction, so drift is a genuine diagnostic, not a tautology); long-time equilibrium ratios
match the input thermodynamics; halving rtol changes reported quantities by less than the 1%
error budget. Now it's a result.

## Failure modes & pitfalls

- **Default-blindness:** reporting `solve_ivp` output at the default `rtol=1e-3` (three digits,
  per step, at best) for a result quoted to 4 digits; MC without error bars; grid resolution
  chosen by available memory rather than a convergence study. Every reported digit must be backed
  by a knob-tightening test.
- **Scalar `atol` across mixed-magnitude state** — silently zeroes the small components (above).
- **The failed-explicit-solver misdiagnosis:** responding to RK45 slowness by loosening tolerances
  (garbage, faster) or shrinking `max_step` (correct-ish, slower) instead of switching to
  BDF/Radau.
- **Float time accumulation:** `t += dt` drifts (especially float32), and `while t != t_end` can
  run forever. Compute `t = i * dt` and compare with a tolerance.
- **`np.random.seed(42)` in library code, or the same seed in every worker:** N "independent"
  replicas of one stream. Use `SeedSequence.spawn`; in JAX, split keys and never reuse one.
- **Unit mixing:** a rate in 1/hour meets a dt in seconds; nothing errors; everything is wrong by
  3600×. Either `pint` at the interfaces or a single-unit-system rule enforced in review — and
  dimension-check the headline number's magnitude before believing it.
- **Validating only the happy path:** conservation laws are *free* tests — mass, energy, charge,
  probability summing to 1, symplectic invariants. Assert them in code, not by eyeball. Limiting
  cases (rate → 0 recovers the simpler model; N → ∞ recovers the ODE) catch structural bugs that
  convergence studies miss.
- **Convergence study done weakly:** "halved the step, answer barely changed" is necessary but
  weak. Check the *order*: for a method of order p, halving h should shrink the error against a
  reference (finest grid or analytic) by ≈ 2^p. Wrong observed order = a bug in the
  discretization or a non-smooth solution violating the method's assumptions. This test catches
  sign errors that "looks converged" never will.
- **JAX-specific:** Python `if` on a traced value (`TracerBoolConversionError` — use `lax.cond`);
  benchmarking without `.block_until_ready()` (async dispatch makes everything look instant);
  forgetting x64 mode and blaming the algorithm for float32 noise.
- **Parallelism before vectorization**, and `multiprocessing` over big arrays without shared
  memory — pickling costs dominate the compute.

## Worked micro-example: stiff solve with tolerance discipline (scipy)

```python
import numpy as np
from scipy.integrate import solve_ivp

# Robertson problem — the canonical stiff-kinetics test (rates span 9 orders of magnitude).
def f(t, y):
    return np.array([-0.04*y[0] + 1e4*y[1]*y[2],
                      0.04*y[0] - 1e4*y[1]*y[2] - 3e7*y[1]**2,
                      3e7*y[1]**2])

def jac(t, y):
    return np.array([[-0.04,  1e4*y[2],            1e4*y[1]],
                     [ 0.04, -1e4*y[2] - 6e7*y[1], -1e4*y[1]],
                     [ 0.0,   6e7*y[1],             0.0     ]])

y0   = [1.0, 0.0, 0.0]
atol = np.array([1e-8, 1e-12, 1e-8])   # y[1] peaks near 1e-5: per-component floor, not scalar
sol  = solve_ivp(f, (0.0, 1e5), y0, method="BDF", jac=jac,
                 rtol=1e-6, atol=atol, dense_output=True)
assert sol.success

# Free validation: the components form a conserved total — assert it, don't eyeball it.
drift = np.abs(sol.y.sum(axis=0) - 1.0).max()
assert drift < 1e-7, f"conservation violated: {drift:.2e}"

# Tolerance check: re-run at rtol=1e-8; reported quantities must move < the error budget.
# (For contrast, RK45 on this problem: minutes of runtime and millions of steps —
#  the failed-explicit-solver signature that says "stiff".)
```

## Verification / self-check

Before presenting any numerical result, confirm all of:
- **Conservation / invariants** asserted in code and passing — use whatever the physics gives
  you for free.
- **Limiting case** reproduced: a parameter at 0/∞ recovers a known simpler answer, or an
  analytic special case matches to the expected digits.
- **Convergence with an order check:** tighten tolerance / halve step / quadruple N — the answer
  moves less than the stated error budget, and the error shrinks at the method's theoretical rate.
- **Reproducibility:** seeds and versions logged; a rerun produces identical output
  (deterministic) or statistically consistent output within stated error bars (stochastic).
- **Units audit** on inputs and on the headline number's dimensional plausibility.

Stopping rules: stop refining numerics when numerical error sits comfortably below *model* error —
parameter uncertainty usually dominates, and a 1%-accurate solve of a model with 20% parameter
error is done. Stop optimizing when runtime fits the iteration loop you actually need — a
40-second simulation run 100 times a day needs no rewrite. Stop climbing the modeling ladder when
the current rung answers the question asked, with error bars.
