---
name: probability-and-statistics
description: Loads when reasoning about uncertainty, probabilities, or data — Bayes/conditioning questions, choosing distributions, interpreting p-values or confidence intervals, designing or critiquing A/B tests, picking estimators, applying CLT-based approximations, bootstrap/permutation methods, or checking statistical claims for classic fallacies.
---

# Probability & Statistics Without the Classic Errors

## Core mental model

1. **P(A|B) ≠ P(B|A), and every famous fallacy is this one error.** Prosecutor's fallacy, base-rate neglect, medical-test panic — all confuse the likelihood of the evidence given a hypothesis with the probability of the hypothesis given the evidence. The correction is always the same mechanical move: bring in the prior via Bayes, ideally in odds form: posterior odds = prior odds × likelihood ratio.
2. **Distributions arise from mechanisms, not vibes.** Choose by asking what process generated the data: counts of independent rare events in a window → Poisson; waiting time between memoryless events → exponential; proportions with pseudo-count evidence → beta; product of many small positive multiplicative effects → lognormal; sum of many comparable independent effects → normal (CLT); max/min of many draws → extreme-value, *not* normal.
3. **The CLT is a license with conditions.** It needs (a) finite variance, (b) enough effective independence, (c) no single term dominating the sum. Heavy tails (Pareto-like with tail index α ≤ 2), strong correlation (time series, users within clusters), or one whale observation void the license — and then "±2 SE" intervals are fiction.
4. **A p-value is P(data at least this extreme | H₀), full stop.** Not P(H₀ | data), not the probability of a fluke, not 1 − P(replication). Most testing landmines come from acting as if it were one of those.
5. **Conditioning defines the question.** "Probability of X" is meaningless until the reference class is fixed; changing what you condition on (selected samples, survivors, observed-only data) silently changes the answer. Selection effects — survivorship, Berkson, winner's curse — are conditioning bugs, not bad luck.
6. **When in doubt, simulate.** Any probability claim you can state, you can Monte Carlo in 10 lines. Simulation is the spell-checker of probabilistic reasoning — run it before presenting any non-trivial derived probability, expectation, or interval.

## Decision frameworks

### Bayes in odds form (do this mechanically)
Given prevalence π, sensitivity `sens`, false-positive rate `fpr`:
posterior odds = [π/(1−π)] × [sens/fpr].
Example: π = 0.001, sens = 0.99, fpr = 0.05 → odds = (1/999)(0.99/0.05) ≈ 0.0198 → P ≈ 1.9%. A "99% accurate" test on a rare condition leaves you near 2%, and any answer near 99% means the prior got dropped. Stack independent evidence by multiplying likelihood ratios — and check independence before you multiply (correlated evidence double-counts).

### Which distribution — by generating mechanism
| Data looks like | Reach for | Because | Check before committing |
|---|---|---|---|
| Event counts per unit time/space | Poisson | independent rare events at rate λ | variance ≈ mean? Real counts usually have var ≫ mean → negative binomial |
| Time-to-event, memoryless | Exponential | constant hazard | hazard actually constant? Wear-out/burn-in → Weibull or gamma |
| Rates and probabilities in [0,1] | Beta | conjugate; α−1 successes, β−1 failures as pseudo-counts | point masses at exactly 0/1 need zero-one inflation |
| Incomes, latencies, file sizes | Lognormal | multiplicative growth; take logs and look | tail heavier than lognormal (straight log-log CCDF) → Pareto-like; sample means unstable |
| Successes in n independent trials | Binomial | fixed n, constant p | clustered/correlated trials → overdispersion → beta-binomial |
| Sums/averages of many comparable terms | Normal | CLT | tails, dependence, domination (see CLT limits) |
| Extremes (max flood, max latency) | GEV / GPD (extreme value theory) | maxima do not obey the CLT | enough tail data to fit; block maxima vs peaks-over-threshold |
| "Time until k-th event" | Gamma / Erlang | sum of exponential waits | same hazard-constancy caveat |

### Testing method selection
- **Default for "is this difference real?"**: permutation test on the statistic you actually care about. Exact under the null of exchangeability, assumption-light, works for medians, ratios, AUCs, anything.
- **Default for "what's the uncertainty on this estimate?"**: bootstrap CI (`scipy.stats.bootstrap`, prefer `method='BCa'`). Works for almost any statistic. Fails for: extremes (min/max, quantiles near 0 or 1), n < ~20, and non-smooth statistics.
- **t/z tests**: fine when n is large and tails are tame; their main modern value is *power analysis before* the experiment, not analysis after.
- **Dependent data** (time series, repeated measures per user): block bootstrap, cluster-robust standard errors, or mixed models. Naive iid resampling of autocorrelated data understates SEs, often by 2–5×.
- **Choosing n**: power analysis first, always. To detect effect d (in SDs) at α=0.05 with 80% power, n per group ≈ 16/d². d = 0.1 → ~1600/group. If the feasible n can't reach power ≥ 0.5, redesign (bigger effect via targeting, paired design, variance reduction via CUPED/regression adjustment) rather than run a doomed test.
- **Bayesian vs frequentist is a tooling choice.** Go Bayesian when: genuine prior information, small n, hierarchical/partial-pooling structure (many small groups → shrink group estimates toward the grand mean), decisions needing utilities. Go frequentist when: you need calibrated error-rate guarantees over repeated use (pipelines, regulatory), or n is large enough that priors wash out. Refuse ideological framing; with flat priors and big n they numerically agree, and the choice should be driven by which machinery fits the structure.

### Simulation patterns (the verification workhorse)
- **Forward simulation**: encode the generative story, run it 10⁵ times, count. Settles arguments about conditional probabilities (Monty Hall, boy-girl paradoxes, birthday variants) in seconds. The discipline is simulating the *exact* conditioning: filter runs by what was actually observed, then measure frequency within the filtered set — most paradoxes are people simulating the wrong filter.
- **Parametric bootstrap**: fit the model, simulate datasets from the fit, re-fit each — the spread of re-fits is your uncertainty *under the model*. Compare with the nonparametric bootstrap; big disagreement means the model's assumptions are doing heavy lifting.
- **Null simulation**: simulate under H₀ with your full analysis pipeline (including any selection/peeking behavior) to get the *real* false-positive rate rather than the nominal one.
- **Power simulation**: simulate under the effect size you'd care about; the fraction of significant results is your power. Ten lines, and it works for any design (clustered, sequential, non-normal) where formulas don't exist.
- Costing: 10⁵ draws gives Monte Carlo SE of a probability estimate ≈ √(p(1−p)/10⁵) ≈ 0.0016 at p = 0.5 — quote probabilities from simulation to at most 2–3 digits and increase draws when tail probabilities (p < 0.01) are the target (you need ~100/p draws for a decent relative error).

### Estimator judgment
- Unbiasedness is overrated; MSE = bias² + variance is what you eat. Shrinkage (James–Stein, ridge, partial pooling) deliberately buys bias to slash variance and wins whenever you estimate many related quantities. Prefer unbiasedness mainly when estimates get *summed downstream* (biases accumulate; variances average out).
- MLE is asymptotically efficient but fragile at edges: p̂ = 0/“0 successes” breaks products and logs — smooth with Laplace/beta pseudo-counts before multiplying. Boundary MLEs (variance components at 0) need profile likelihood or Bayes.
- Medians and trimmed means are not "weaker t-tests" — under heavy tails they have *lower* variance than the mean. Match the estimator to the tail, not to convention.
- Report uncertainty of the estimator you'll actually use (e.g., the ratio of two means needs the delta method or bootstrap — not the SEs of numerator and denominator separately).

### What to report: point, interval, or distribution
- Repeatable, symmetric-loss, aggregated decision → point estimate with SE suffices.
- One-shot decision or asymmetric loss (capacity planning, risk) → quantiles and exceedance probabilities; the mean is a distraction.
- Estimate feeds a nonlinear downstream computation → propagate the whole distribution (simulate) — E[f(X)] ≠ f(E[X]) for nonlinear f (Jensen: the direction of the bias follows the curvature; log and ratios are the everyday offenders).
- Ratio-of-estimates uncertainty → delta method (Var[g(θ̂)] ≈ g′(θ̂)²·Var[θ̂]) or bootstrap the ratio directly; never divide the point estimates and reuse the numerator's CI.
- Many related small-sample estimates (per-segment rates, per-item scores) → hierarchical shrinkage or at least beta-binomial smoothing; raw per-group MLEs will be dominated by the smallest groups at both extremes of the leaderboard.

## Failure modes & pitfalls

- **Base-rate neglect in any classifier/test discussion.** Sensitivity and specificity are properties of the test; precision (PPV) depends on prevalence. Never say "the model is 95% accurate so this flagged item is 95% likely bad" — compute PPV at the deployment base rate. At 0.1% prevalence, a 95%-specific flagger yields ~50 false positives per true positive.
- **Prosecutor's fallacy — and its mirror.** "Match probability 1 in a million → 1-in-a-million innocent" ignores the population of possible matches. The defense-attorney inverse ("8M people → 8 suspects → only 1/8") ignores that other evidence already shrank the reference class. Likelihood ratios must multiply the *correct* prior, in both directions.
- **"p = 0.03 means 3% chance the null is true."** No. With low prior odds of a real effect and modest power, a majority of p < 0.05 "discoveries" can be false — the false discovery rate depends on prior and power, not just α. FDR ≈ π₀·α / (π₀·α + (1−π₀)·power) — plug numbers before celebrating.
- **Optional stopping ("peek until significant").** Repeated testing at fixed α = 0.05 as data accrues drives false-positive rates toward 20–30%+ over many peeks. Fixes: fix n in advance; group-sequential alpha-spending (O'Brien–Fleming); or always-valid inference (confidence sequences / mSPRT) built for continuous monitoring. Dashboards that re-test daily are optional stopping wearing a suit.
- **Multiple comparisons hiding in the forking paths.** 20 metrics × 5 segments × 3 variants = 300 implicit tests; something *will* clear p < 0.05. Fixes: Benjamini–Hochberg for FDR control in exploration; Holm/Bonferroni for strict confirmation; best, pre-declare one primary metric and demote the rest to descriptive.
- **Declaring "no effect" from a non-significant test.** p = 0.4 at n = 30 usually means "we couldn't detect anything smaller than a huge effect". Report the CI (it says what's ruled out) or run an equivalence test (TOST). Absence of evidence ≠ evidence of absence — mechanically, not rhetorically.
- **Applying the CLT to heavy-tailed averages.** Means of Pareto(α ≤ 2) data don't stabilize — the sum is dominated by the largest observation, and "mean latency ± SE" is fiction. Use quantiles, log-transform, or fit the tail. Diagnostic: does the mean move materially when you delete the single largest point?
- **Treating correlated rows as independent n.** 10,000 pageviews from 500 users is n ≈ 500 (or fewer) for user-level questions. Design effect ≈ 1 + (m−1)ρ: cluster size m = 20 with intra-class correlation ρ = 0.05 doubles the variance. Randomize at the unit you analyze (user-level randomization for user-level metrics).
- **Regression to the mean read as causation.** Selecting extremes (worst performers, highest-error items) guarantees improvement on remeasurement with zero intervention. Any before/after claim on a selected-because-extreme group needs a control group — full stop.
- **Winner's curse in reported effects.** The variants that clear a significance bar have effect sizes biased upward (selection on noise). Expect shrinkage on replication; plan capacity to the CI's lower bound, not the point estimate.
- **Survivorship and Berkson.** Analyses of "companies that survived", "requests that completed", "patients admitted" condition on an outcome-correlated event; correlations flip sign this way (Berkson: conditioning on a collider induces spurious negative correlation between its causes). First question for any observational dataset: what had to happen for a row to exist?
- **Simpson's paradox in pooled comparisons.** Aggregate rates can reverse every stratum's ordering when group mixes differ. Before comparing two observational rates, stratify once by the obvious confounder (segment, time, severity).
- **Bootstrap misuse:** bootstrapping the max/min, iid-resampling a time series, bootstrapping n = 8, or quoting a bootstrap SE for a bimodal statistic. And percentile intervals aren't automatically better for skewed statistics at small n — prefer BCa.
- **CI misreading.** "95% CI [2,5]" is a statement about the *procedure's* long-run capture rate, not P(θ ∈ [2,5]) = 0.95 — the latter needs a prior and a credible interval. Numerically similar in easy problems; the distinction matters when decisions stack on the tail.
- **Testing hypotheses on the data that suggested them.** Exploratory finding → confirmatory test on the *same* data is circular; the p-value has no meaning. Split the data, or collect fresh data for confirmation.
- **Independence assumed because it's convenient.** Multiplying probabilities (failure rates of "independent" components sharing a power supply; "independent" fraud signals derived from the same field) understates joint tail risk by orders of magnitude. Before multiplying, name the mechanism that would make the events dependent; if you can name one, estimate the joint directly or bound it.
- **Confusing likelihood with probability of the data model.** "The data are unlikely under H₀" says nothing by itself — data can be unlikely under *every* hypothesis (likelihoods are densities, not probabilities). Only likelihood *ratios* between live hypotheses carry evidential weight.
- **Quoting expected value where a distributional statement is needed.** "Expected cost is $50k" is compatible with "5% chance of $1M". For decisions with asymmetric loss, report quantiles or exceedance probabilities; expectation alone is only sufficient when the decision-maker is risk-neutral *and* the bet repeats many times.
- **Overfitting the prior to the answer you want.** A Bayesian analysis whose conclusion flips under any reasonable alternative prior is a prior report, not a data analysis. Sensitivity-check: rerun with a flat, a skeptical, and an enthusiastic prior; report the range.
- **Conflating statistical and practical significance.** At n = 10⁷, a 0.01% difference clears any α; significance says "measurable", not "matters". Lead with the effect size and its CI; the p-value is a footnote at large n.
- **Standard deviation vs standard error confusion in reporting.** SD describes the population spread (doesn't shrink with n); SE describes the estimate's precision (shrinks as 1/√n). Error bars must be labeled; readers systematically misread SD bars as uncertainty and SE bars as spread.
- **Percentiles of aggregates vs aggregates of percentiles.** The p99 of per-server latencies averaged across servers is not the fleet p99; percentiles don't average, sum, or subtract. Recompute quantiles on the pooled distribution (or use mergeable sketches: t-digest, DDSketch).
- **Zero-count cells breaking rate ratios.** 0 events in arm A → naive relative risk of 0 or ∞ with absurd CIs. Use exact/mid-p intervals or add continuity corrections deliberately — and say so — rather than letting a downstream division silently produce inf.

## Worked micro-examples

**1. Optional stopping, quantified by simulation (the verification habit).**
```python
import numpy as np
rng = np.random.default_rng(0)
hits = 0
for _ in range(2000):
    x = rng.normal(0, 1, 1000)          # H0 true
    for n in range(100, 1001, 100):     # peek every 100 samples
        z = x[:n].mean() / (x[:n].std(ddof=1) / np.sqrt(n))
        if abs(z) > 1.96:
            hits += 1; break
print(hits / 2000)   # ~0.17–0.20, not 0.05
```
Ten peeks roughly quadruple the false-positive rate. Show the simulation instead of arguing.

**2. Permutation test as the assumption-light default.**
```python
def perm_test(a, b, stat=np.median, n=10_000, rng=np.random.default_rng(0)):
    obs = stat(a) - stat(b)
    pooled = np.concatenate([a, b])
    count = 0
    for _ in range(n):
        p = rng.permutation(pooled)
        count += abs(stat(p[:len(a)]) - stat(p[len(a):])) >= abs(obs)
    return (count + 1) / (n + 1)     # add-one: never report p = 0
```
Works for medians, trimmed means, ratios — anything computable. The exchangeability assumption fails for time series and clusters; permute *blocks or clusters* there instead of rows.

**3. Beta-binomial ranking (the "sort by average rating" fix).**
Item A: 9/10 positive. Item B: 90/100. Naive rates tie at 0.90. With a Beta(1,1) prior, posterior means: A = 10/12 ≈ 0.833, B = 91/102 ≈ 0.892 — B ranks higher, matching the intuition that 100 trials is stronger evidence. Risk-averse ranking uses a lower posterior quantile: `scipy.stats.beta.ppf(0.05, 1+k, 1+n-k)` gives A → 0.61, B → 0.84. This one move fixes most "5-review item tops the leaderboard" bugs, and the prior strength (pseudo-count total α+β) is the only knob.

**4. Jensen bite in a routine metric.**
Per-user conversion rates rᵢ are averaged, then someone reports "average time between conversions = 1/mean(r)". With r = [0.01, 0.1, 0.5] : 1/mean(r) = 1/0.2033 ≈ 4.9, but mean(1/r) = (100 + 10 + 2)/3 ≈ 37.3 — off by 7.6×. E[1/X] ≥ 1/E[X] (Jensen, convex 1/x), and the gap explodes with heterogeneity. Any pipeline that swaps the order of a nonlinear transform and an average needs a simulation check; harmonic-vs-arithmetic mean bugs in latency ("average rate") reporting are this exact error.

**5. Design effect arithmetic.**
An experiment logs 40,000 sessions from 2,000 users (m = 20 sessions/user), session-level metric with intra-user correlation ρ = 0.1. Design effect = 1 + 19×0.1 = 2.9 → effective n ≈ 40,000/2.9 ≈ 13,800, and the naive SE is √2.9 ≈ 1.7× too small — enough to turn p = 0.05 into p ≈ 0.24. Any session-level analysis of user-randomized data must widen SEs by this factor or cluster on user.

## Verification & self-check

- **Simulate before you ship**: for any derived probability, expectation, or interval, write a 10-line Monte Carlo and confirm the analytic answer to ~2 digits. When they disagree, the derivation is wrong more often than the simulation.
- **Unit-test the null**: run the full analysis pipeline on data where the effect is known to be zero (shuffled labels). It must flag at exactly its nominal α. A pipeline that "finds" effects in 20% of shuffles at α = 0.05 is the discovery, not the data.
- **Sanity-check the implied likelihood ratio**: if a posterior moved from 0.1% to 90%, the evidence claimed an LR of ~9000:1 — is it really that diagnostic?
- **State n_effective, not n**: any clustering, autocorrelation, or weighting shrinks effective sample size; compute and report it.
- **Tail audit before summarizing**: one log-CCDF or QQ plot before trusting any mean ± SD; it prevents the entire heavy-tail error class.
- **Row-existence audit**: describe the process that put each row in the dataset; name the selection events you're conditioning on.
- **Ask "what's the base rate?" out loud** whenever converting test/classifier characteristics into a probability about an individual case.
- **Probabilities must obey the axioms**: check that reported probabilities over exhaustive outcomes sum to 1, that P(A and B) ≤ min(P(A), P(B)), and that no conditioning step increased a probability without a mechanism. Violations of these cheap identities catch a surprising fraction of multi-step probability derivations.
- **Calibration over confidence**: for any probabilistic forecast pipeline, bin predictions by stated probability and compare against realized frequencies (reliability curve). A model that says "90%" and is right 70% of the time is miscalibrated regardless of its AUC.
