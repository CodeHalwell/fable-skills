---
name: simulation-and-scientific-computing
description: Load when building or debugging numerical simulations and scientific computation — ODE/PDE solving, Monte Carlo methods, stochastic models, scipy/NumPy/JAX numerical code, solver selection and tolerances, reproducibility with random seeds, unit handling, or validating simulation correctness and performance.
---

# Simulation & Scientific Computing

Audit finding: frontier models hold essentially all of this cold — stiffness diagnosis, LSODA triage, per-component atol, 1/√N economics, importance sampling + ESS collapse, `SeedSequence.spawn`, the JAX ecosystem and x64 gotcha, order-checked convergence, event detection. This file is a compressed review checklist; trust the model's derivations and use this to ensure nothing is skipped.

## Decision anchors (one line each)

1. Climb the modeling ladder from the bottom (analytic > ODE > PDE > agent/MC; each rung 10–1000× costlier); the rungs break upward on nonlinearity/heterogeneity, discreteness/fluctuations, and distribution-shaped questions respectively.
2. Explicit solver grinding millions of tiny steps on a smooth solution = stiffness; switch family (BDF/Radau, Jacobian supplied), don't tune tolerances. LSODA much faster than RK45 *is* the diagnosis.
3. `atol` is per-component: state spanning 1e-12..1e-3 with scalar `atol=1e-6` makes trace species pure noise — pass a vector scaled to each component, then verify by tightening 10× and checking the answer moves less than the error budget.
4. Events via `events=` root-finding, never step-and-check.
5. MC results carry standard errors or they're meaningless; a digit costs 100×, so the answer to "unaffordable N" is variance reduction (antithetic → control variates → importance sampling for rare events → scrambled Sobol QMC), not patience.
6. Seeds: `SeedSequence(base).spawn(n)` — reproducible AND independent; `seed+rank` is the folklore anti-pattern.
7. Vectorize before parallelize before rewrite (~100× interpreter tax; parallelizing interpreted Python buys core-count× against it). Profile → vectorize → numba/JAX jit for sequential recurrences → vmap/multiprocessing → compiled rewrite.
8. JAX when you need gradients through the simulation (diffrax solve replaces finite-difference sweeps), vmap batch parallelism, or accelerators; ecosystem = diffrax/equinox/NumPyro/optax/BlackJAX; `jax_enable_x64` first or chase phantom float32 bugs.
9. Deterministic for large-population means; stochastic when discreteness, distributions, or noise-feedback matter — and build the deterministic model first anyway: it's the debugging baseline, a control variate, and the large-N limit the stochastic mean must recover (Gillespie vs mass-action ODE is the canonical validation pair).
10. Units are a correctness class: SI-everywhere or `pint` at boundaries, unit-suffixed names (`dt_s`), and dimension-check the headline number.

## Pitfall checklist (the ones cold models don't volunteer unprompted)

- Responding to RK45 slowness by loosening tolerances (garbage, faster) or shrinking `max_step` (slower) instead of switching method family.
- "Noise" in a deterministic model's output = the solver bouncing along its stability boundary, not stochasticity — same stiffness fix.
- Float time accumulation: `t += dt` drifts and `while t != t_end` can run forever — compute `t = i * dt`, compare with tolerance.
- Rejecting a rewrite the numbers don't justify: a 40-second solve inside the iteration budget needs no Julia/JAX port — runtime stopping rule beats enthusiasm.
- JAX: Python `if` on traced values (`lax.cond`); benchmarking without `.block_until_ready()` (async dispatch makes everything look instant); reusing PRNG keys.
- `multiprocessing` over big arrays without shared memory — pickling eats the gain.
- Grid resolution chosen by available memory rather than a convergence study; every reported digit needs a knob-tightening test behind it.

## Verification (the definition of done, not a final step)

- **Conservation/invariants asserted in code** (mass, energy, probability, symplectic invariants) — free tests the physics gives you. Note the useful subtlety: BDF isn't conservative by construction, so small drift is a genuine diagnostic, not a tautology.
- **Limiting cases**: parameter → 0/∞ recovers the simpler model; stochastic mean → ODE at large N. Catches structural bugs convergence studies miss.
- **Order-checked convergence**: for method order p, halving h shrinks error ≈ 2^p against a reference. Wrong observed order = discretization bug or violated smoothness assumption — this catches sign errors that "looks converged" never will.
- **Reproducibility**: seeds + versions logged; reruns identical (deterministic) or within stated error bars (stochastic).
- **Units audit** on inputs and the headline number's dimensional plausibility.

Stopping rules: stop refining numerics when numerical error sits comfortably below *model* error (a 1%-accurate solve of a 20%-uncertain model is done); stop optimizing when runtime fits the actual iteration loop; stop climbing the ladder when the current rung answers the question asked, with error bars.

## Worked micro-example: stiff solve with tolerance discipline (kept minimal)

```python
# Robertson problem — rates span 9 orders of magnitude; RK45 takes minutes/millions of steps.
atol = np.array([1e-8, 1e-12, 1e-8])          # y[1] peaks near 1e-5: per-component floor
sol = solve_ivp(f, (0, 1e5), [1.0, 0.0, 0.0], method="BDF", jac=jac,
                rtol=1e-6, atol=atol, dense_output=True)
assert np.abs(sol.y.sum(axis=0) - 1.0).max() < 1e-7   # conservation asserted, not eyeballed
# Then re-run at rtol=1e-8: reported quantities must move < the error budget.
```

The expert sequence it encodes: diagnose stiffness from the failed-explicit signature → switch family + supply the Jacobian (6 h → 40 s) → *don't stop* — the default scalar atol was still lying about trace species (they moved 30% under tightening: the earlier trace results were artifact) → validate with conservation, equilibrium thermodynamics, and a tolerance sweep → reject the rewrite (40 s is inside the iteration budget).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 12 baseline, 0 partial, 0 delta — among the strongest baselines audited.
- Opus cold nails: stiffness diagnosis + BDF/Radau/LSODA triage, vector atol + tightening verification, 1/√N + variance-reduction ladder + 1e8-draws rare-event arithmetic + ESS-collapse diagnostic, SeedSequence.spawn vs seed+rank, JAX ecosystem + x64 gotcha, profile→vectorize→jit ordering, order-checked convergence, Gillespie-vs-ODE validation, event detection.
- Restructured from survey to checklist; retained value = the pitfalls it doesn't volunteer (tolerance-loosening misresponse, float-time accumulation, block_until_ready, stopping rules) and the compressed expert sequence.
