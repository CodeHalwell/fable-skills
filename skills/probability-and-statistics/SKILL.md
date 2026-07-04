---
name: probability-and-statistics
description: Loads when reasoning about uncertainty, probabilities, or data — Bayes/conditioning questions, choosing distributions, interpreting p-values or confidence intervals, designing or critiquing A/B tests, picking estimators, applying CLT-based approximations, bootstrap/permutation methods, or checking statistical claims for classic fallacies.
---

# Probability & Statistics Without the Classic Errors

## Core mental model

1. **P(A|B) ≠ P(B|A), and every famous fallacy is this one error.** Prosecutor's fallacy, base-rate neglect, medical-test panic — all confuse the likelihood of evidence given a hypothesis with the probability of the hypothesis given evidence. The correction is always the same mechanical move: bring in the prior via Bayes, ideally in odds form: posterior odds = prior odds × likelihood ratio.
2. **Distributions arise from mechanisms, not vibes.** Choose a distribution by asking what process generated the data: counts of independent rare events in a window → Poisson; waiting time between memoryless events → exponential; proportions/probabilities with pseudo-counts → beta; product of many small positive multiplicative effects → lognormal; sums of many comparable independent effects → normal (CLT); max/min of many draws → extreme-value, *not* normal.
3. **The CLT is a license with conditions.** It needs (a) finite variance, (b) enough effective independence, (c) no single term dominating the sum. Heavy tails (Pareto α ≤ 2), strong correlation (time series, users within clusters), or one whale observation void the license — your "±2 SE" interval is then fiction.
4. **A p-value is P(data at least this extreme | H₀), full stop.** It is not P(H₀|data), not the probability of a fluke, not 1 − P(replication). Most testing landmines come from acting as if it were one of those.
5. **When in doubt, simulate.** Any probability claim you can state, you can Monte Carlo in 10 lines. Simulation is the spell-checker of probabilistic reasoning — use it before presenting any non-trivial derived probability, expectation, or interval.

## Decision frameworks

### Bayes in odds form (do this mechanically)
Given prevalence π, sensitivity `sens`, false-positive rate `fpr`:
posterior odds = [π/(1−π)] × [sens/fpr]. Example: π = 0.001, sens = 0.99, fpr = 0.05 → odds = (1/999)(0.99/0.05) ≈ 0.0198 → P ≈ 1.9%. A "99% accurate" test on a rare condition leaves you at ~2%, and any answer near 99% means the prior was dropped.

### Which distribution — by generating mechanism
| Data looks like | Reach for | Because | Check before committing |
|---|---|---|---|
| Event counts per unit time/space | Poisson | independent rare events, rate λ | variance ≈ mean? If var ≫ mean (common!), use negative binomial |
| Time-to-event, memoryless | Exponential | constant hazard | hazard actually constant? Wear-out or burn-in → Weibull/gamma |
| Rates, probabilities in [0,1] | Beta | conjugate pseudo-counts α−1 successes, β−1 failures | boundary spikes at exactly 0/1 need zero-one inflation |
| Incomes, latencies, file sizes | Lognormal | multiplicative growth; log it and look | tail heavier than lognormal (straight line on log-log CCDF) → Pareto-like; means may be unstable |
| Successes in n independent trials | Binomial | fixed n, constant p | clustered trials → overdispersion → beta-binomial |
| Sums/averages of many comparable terms | Normal | CLT | tails, dependence, domination (see CLT limits) |

### Testing method selection
- **Default for "is this difference real?"**: permutation test on the statistic you actually care about. Exact under the null of exchangeability, assumption-light, works for medians/ratios/AUCs.
- **Default for "what's the uncertainty on this estimate?"**: bootstrap percentile or BCa CI (`scipy.stats.bootstrap`). Works for almost any statistic. Fails for: extremes (min/max/quantiles near 0 or 1), sample sizes < ~20, and statistics that are non-smooth functions of the data.
- **t-test/z-test**: fine when n is large and tails are tame; the real value of the classic tests today is power *analysis* before the experiment, not analysis after.
- **Dependent data** (time series, repeated measures per user): block bootstrap / cluster-robust errors / mixed models. Naive iid resampling of autocorrelated data understates SEs, often by 2–5×.
- **Bayesian vs frequentist is a tooling choice.** Go Bayesian when: you have real prior information, small n, hierarchical/partial-pooling structure, or you need decisions with utilities. Go frequentist when: you need calibrated error-rate guarantees across repeated use (pipelines, regulatory), or n is large enough that the prior is irrelevant anyway. Refuse to frame it as ideology.

### Estimator judgment
- Unbiasedness is overrated; MSE = bias² + variance is what you eat. James–Stein/shrinkage and ridge deliberately buy bias to slash variance. Prefer a slightly biased low-variance estimator for prediction; prefer unbiasedness when estimates will be *summed/averaged downstream* (biases add).
- MLE is asymptotically efficient but fragile at edges: MLE of variance divides by n (biased, fine); MLE with a parameter on the boundary (p̂=0 from 0 successes) needs smoothing (Laplace/beta prior) before you put it in a product or log.
- Medians and trimmed means are not "less powerful t-tests" — with heavy tails they have *lower* variance than the mean. Match the estimator to the tail.

## Failure modes & pitfalls

- **Base-rate neglect in any classifier/test discussion.** Precision depends on prevalence; sensitivity/specificity don't. Never quote "the model is 95% accurate so this flagged item is 95% likely bad" — compute PPV at the deployment base rate. At 0.1% prevalence, a 95%-specific flagger produces ~50 false positives per true positive.
- **Prosecutor's fallacy in reverse (defense attorney's fallacy) too:** "the match probability is 1 in a million and there are 8M people in the city, so 8 suspects, so 1/8" — this ignores that the other evidence already restricted the reference class. Likelihood ratios must be applied to the *correct prior*, in both directions.
- **p = 0.03 means "3% chance the null is true."** No. With low prior odds of a real effect and modest power, most p<0.05 findings can still be false — that's the false discovery rate, which depends on prior and power, not just α.
- **Optional stopping ("peek until significant").** Testing repeatedly as data accrues at fixed α=0.05 pushes false-positive rates toward 30%+ over many peeks. Corrections: fix n in advance, use group-sequential alpha-spending (O'Brien–Fleming), or use always-valid methods (mixture sequential probability ratio tests / confidence sequences) designed for continuous monitoring.
- **Multiple comparisons hiding in the garden of forking paths.** 20 metrics, 5 segments, 3 model variants = 300 implicit tests; something will hit p<0.05. Corrections: Benjamini–Hochberg for FDR control when exploring, Bonferroni/Holm for strict confirmation, or (best) declare one primary metric pre-analysis and treat the rest as descriptive.
- **Declaring "no effect" from a non-significant test with no power analysis.** p=0.4 with n=30 typically means "we couldn't have detected anything under a 1-SD effect anyway." Either report the CI (which quantifies what's ruled out) or do an equivalence test (TOST); never convert absence of evidence into evidence of absence.
- **Applying the CLT to averages of heavy-tailed data.** Sample means of Pareto(α=1.5) data have infinite variance — the "mean latency ± SE" of a fat-tailed latency distribution is dominated by the biggest observation and doesn't stabilize. Use quantiles, log-transform, or report the tail exponent.
- **Treating correlated rows as independent n.** 10,000 pageviews from 500 users is n≈500 (or less) for anything user-level. Compute the design effect ≈ 1 + (m−1)ρ (m = cluster size, ρ = intra-class correlation); a mere ρ=0.05 with m=20 doubles your variance.
- **Continuous-monitoring dashboards re-testing daily** are optional stopping wearing a suit. Same fix.
- **Simpson's paradox in any pooled comparison.** Aggregate rates can reverse every stratum's ordering when group sizes differ across strata. Before comparing two rates from observational data, always break out by the obvious confounder (segment, time, severity mix) once.
- **Bootstrap misuse:** bootstrapping the maximum, bootstrapping time series iid, bootstrapping with n=8, or reporting a bootstrap SE for a statistic whose distribution is bimodal. Also: percentile intervals are not automatically better than normal-approx ones for skewed statistics at small n — prefer BCa.
- **Confusing the CI's confidence with a probability statement about the parameter.** "95% CI [2,5]" means the *procedure* traps the truth 95% of the time; if you want "P(parameter in [2,5]) = 0.95" you need a Bayesian credible interval and a prior. Usually the practical numbers are close; the interpretation error matters when someone stacks decisions on it.

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
Ten peeks roughly quadruple the false-positive rate. Show this simulation instead of arguing.

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
Works for medians, trimmed means, ratios — anything. The exchangeability assumption fails for time series and clustered data; permute *blocks/clusters* then.

**3. Beta-binomial ranking (the "sort by average rating" fix).**
Item A: 9/10 positive. Item B: 90/100. Naive rates: 0.90 = 0.90. With Beta(1,1) prior, posterior means: A = 10/12 ≈ 0.833, B = 91/102 ≈ 0.892 — B ranks higher, matching intuition that 100 trials is stronger evidence. For risk-averse ranking use a lower posterior quantile (`scipy.stats.beta.ppf(0.05, 1+k, 1+n-k)`): A → 0.61, B → 0.84. This one move fixes most "small-sample item looks best" leaderboard bugs.

## Verification & self-check

- **Simulate before you ship**: any derived probability, expected value, or interval — write a 10-line Monte Carlo and confirm the analytic answer to ~2 digits. Disagreement means the derivation is wrong more often than the simulation.
- **Sanity-check magnitudes against the prior**: if a posterior probability moved from 0.1% to 90% on one weak signal, the likelihood ratio implied is 9000:1 — is the evidence really that strong?
- **Unit test the null**: run your full analysis pipeline on data where the effect is known to be zero (shuffled labels). It should "find" effects at exactly its nominal α. If it flags 20% of shuffles at α=0.05, the pipeline (not the data) is the discovery.
- **Check n_effective, not n**: any clustering, autocorrelation, or reweighting shrinks effective sample size. State it explicitly.
- **Tail audit**: plot log-CCDF or a QQ plot before trusting any mean/SD summary; one plot prevents the heavy-tail class of errors.
- **Ask "what's the base rate?" out loud** whenever converting test/classifier characteristics into a probability about an individual case.
