---
name: probability-and-statistics
description: Loads when reasoning about uncertainty, probabilities, or data — Bayes/conditioning questions, choosing distributions, interpreting p-values or confidence intervals, designing or critiquing A/B tests, picking estimators, applying CLT-based approximations, bootstrap/permutation methods, or checking statistical claims for classic fallacies.
---

# Probability & Statistics Without the Classic Errors

Most of this domain is strong-model baseline (Bayes mechanics, power formulas, design effects, bootstrap failure modes, selection biases). This sheet keeps the working checklists, the numeric anchors, and the verification habits that distinguish a done analysis from a plausible one.

## Working anchors (state these with numbers, not vibes)

- Bayes in odds form, mechanically: posterior odds = [π/(1−π)] × [sens/fpr]. π=0.001, sens=0.99, fpr=0.05 → ≈1.9%. Any answer near 99% dropped the prior. Stack evidence by multiplying LRs — after checking independence (correlated evidence double-counts).
- Power: n per group ≈ 16/d² (α=0.05, 80% power); d=0.1 → ~1600/group. If feasible n can't reach power ≥ 0.5, redesign (targeting, pairing, CUPED/regression adjustment) — don't run a doomed test.
- Peeking: ~10 sequential looks at α=0.05 → real false-positive rate ~20–40%. Fixes: fixed n; O'Brien–Fleming alpha-spending; always-valid inference (confidence sequences/mSPRT) for dashboards that monitor continuously.
- Clustering: design effect = 1 + (m−1)ρ. 20 sessions/user at ρ=0.1 → 2.9; naive SE is √2.9 ≈ 1.7× too small — enough to turn p=0.05 into p≈0.2. Randomize at the unit you analyze, or cluster SEs on it.
- FDR of "discoveries": π₀α / (π₀α + (1−π₀)·power). Plug numbers before celebrating any p<0.05.
- Monte Carlo costing: you need ~100 *events* for ~10% relative error → ~100/p draws; at 10⁵ draws, SE of a p≈0.5 estimate is ~0.0016 — quote simulated probabilities to 2–3 digits.
- Heavy tails: Pareto tail index α ≤ 2 → sample variance (and "±SE") is fiction; α ≤ 1 → the mean itself diverges. Diagnostic that costs one line: does the mean move materially when you delete the single largest point?

## Distribution-by-mechanism checklist (one-liners)

Counts of rare events → Poisson (but check var≈mean; real counts are usually overdispersed → negative binomial). Memoryless waits → exponential (wear-out/burn-in → Weibull/gamma). Rates in [0,1] → beta (pseudo-counts). Multiplicative growth → lognormal (straight log-log CCDF → Pareto instead). Fixed-n trials → binomial (clustered → beta-binomial). Sums of comparable terms → normal, only with finite variance + independence + no dominant term. Extremes → GEV/GPD, never normal.

## Method selection (one-liners)

- "Is the difference real?" → permutation test on the statistic you care about; permute blocks/clusters, not rows, for dependent data.
- "Uncertainty of this estimate?" → bootstrap BCa; fails on extremes, n<~20, non-smooth statistics, and iid-resampled time series (block bootstrap there).
- Ratio of estimates → delta method (keep the covariance term) or bootstrap the ratio; Fieller when the denominator's CV ≳ 0.1.
- Many small related estimates (per-segment rates) → hierarchical shrinkage or beta-binomial smoothing; raw MLEs put the smallest groups at both ends of every leaderboard. Rank risk-averse by a lower posterior quantile: `beta.ppf(0.05, 1+k, 1+n−k)`.
- Bayesian vs frequentist is tooling, not ideology: Bayes for genuine priors, small n, hierarchy, utility decisions; frequentist for calibrated repeated-use error rates. Prior sensitivity check is mandatory: flat/skeptical/enthusiastic priors — a conclusion that flips is a prior report, not an analysis.
- Unbiasedness is overrated (MSE = bias² + variance; shrinkage wins for many related quantities) — *except* when estimates get summed downstream: biases accumulate, variances average out.
- Report quantiles/exceedance probabilities, not expectations, for one-shot or asymmetric-loss decisions; E[f(X)] ≠ f(E[X]) — propagate the distribution through nonlinear downstream computations by simulation.

## Pitfall checklist (compressed; completeness is the point)

Base-rate neglect (PPV depends on prevalence) · prosecutor's fallacy and its defense-attorney mirror · p-value ≠ P(H₀|data) · optional stopping · forking paths (20 metrics × 5 segments = 300 tests; pre-declare one primary) · "no effect" from a non-significant test (report the CI or TOST) · CLT on heavy tails · correlated rows counted as independent n · regression to the mean on selected extremes (needs a control group, full stop) · winner's curse (plan to the CI's lower bound) · survivorship/Berkson (ask what had to happen for a row to exist) · Simpson's (stratify by the obvious confounder once) · CI ≠ credible interval · testing hypotheses on the data that suggested them · independence assumed because convenient (name the mechanism that would couple the events before multiplying) · likelihoods aren't probabilities — only ratios between live hypotheses carry weight · SD vs SE in error bars · percentiles don't average — pool the distribution or use mergeable sketches (t-digest/DDSketch) · zero-count cells → exact/mid-p intervals, not silent inf.

Two that go beyond the standard list:
- **Jensen bites in routine metrics**: 1/mean(r) vs mean(1/r) differ 7.6× for r=[0.01,0.1,0.5]; any pipeline that swaps a nonlinear transform with an average needs a simulation check. Harmonic-vs-arithmetic "average rate" latency bugs are this exact error.
- **Smoothing before multiplying**: MLE p̂=0 breaks products and logs; Laplace/beta pseudo-counts first. Boundary MLEs (variance components at 0) need profile likelihood or Bayes.

## Verification & self-check (the part that actually changes behavior)

- **Simulate before you ship**: any derived probability/expectation/interval gets a 10-line Monte Carlo; disagreement means the derivation is wrong more often than the simulation. For conditional-probability puzzles, simulate the *exact* conditioning: filter runs by what was actually observed, then count within the filtered set — most paradox errors are the wrong filter.
- **Unit-test the null**: run the full pipeline (including any peeking/selection behavior) on shuffled labels; it must flag at its nominal α. A pipeline that finds effects in 20% of shuffles is the discovery.
- **Permutation p-values**: report (count+1)/(n+1), never p=0.
- **Implied-LR sanity check**: a posterior that moved 0.1% → 90% claimed an LR of ~9000:1 — is the evidence really that diagnostic?
- **State n_effective, not n**; do a tail audit (log-CCDF or QQ) before any mean ± SD; do a row-existence audit (name the selection events) before any observational claim.
- **Axioms as cheap tests**: outcome probabilities sum to 1; P(A∧B) ≤ min(P(A),P(B)); no conditioning step raises a probability without a mechanism.
- **Calibration over confidence**: bin forecasts by stated probability vs realized frequency; "90%" that hits 70% is miscalibrated regardless of AUC.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 13 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed everything probed — odds-form Bayes with the 1.9% answer, 16/d², design effect 2.9/1.7×, FDR formula, bootstrap failure list with BCa, Wilson/Beta ranking with worked numbers, quantile-merging via t-digest/DDSketch, delta method with covariance and Fieller, 100/p Monte Carlo rule, prior sensitivity — often with additions (e-values, Hill estimator, immortal-time bias).
- Retained value: the verification habits (simulate the exact filter, null-shuffle the full pipeline), Jensen-in-metrics anchor, and checklist completeness; the exposition was dead weight and was cut.
