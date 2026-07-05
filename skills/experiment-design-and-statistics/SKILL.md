---
name: experiment-design-and-statistics
description: Use for designing experiments that yield decisions — defining the decision first, power analysis before data collection, randomization and blocking, control/counterfactual selection, A/B testing landmines (peeking, novelty effects, interference, multiple metrics), confounders vs colliders and causal-graph reasoning, effect size over p-values, and ML ablation design. Load when planning an experiment, A/B test, or ablation, or when interpreting whether a result supports a decision.
---

# Experiment design and statistics

## Core mental model

1. **Design backward from the decision.** Before anything, state the decision the experiment will inform and the threshold that flips it: "If lift ≥ X% we ship, otherwise we don't." An experiment that can't change any action is not worth running. The decision determines the effect size that matters, which determines the sample size — not the other way around.
2. **Power is a pre-condition, not a post-mortem.** Compute the sample size needed to detect the *smallest effect you care about* (the MDE — minimum detectable effect) at your α and desired power (commonly 80–90%) *before* collecting data. Post-hoc power computed from the observed effect is meaningless. An underpowered experiment mostly produces false negatives and inflated, unreliable "wins."
3. **A control is a counterfactual.** The comparison must answer "what would have happened without the treatment, to the same units, in the same conditions." Randomization makes treatment and control exchangeable in expectation, neutralizing *all* confounders including unknown ones. That is why randomization beats statistical adjustment.
4. **Not all variables should be controlled for.** Causal structure decides. Controlling for a confounder (common cause) removes bias; controlling for a collider (common effect) or a mediator *creates* bias. "Adjust for everything" is wrong. Draw the causal graph first.
5. **Effect size and its uncertainty are the result; the p-value is a footnote.** Report the estimated effect with a confidence interval and judge it against practical significance. "p < 0.05" with a trivial effect is not a finding; a meaningful effect with a wide interval crossing zero is not yet actionable.

## Decision frameworks

- **Sizing the experiment:** fix α (false-positive rate) and power (1 − β). Decide the MDE from the decision threshold (the smallest effect that would change what you do). Estimate baseline variance/rate. Then n follows. For a two-sample mean comparison, n per arm ≈ 16·σ²/Δ² for 80% power at α=0.05 (Δ = MDE) — the 16 encodes (z_{0.975}+z_{0.80})² ≈ (1.96+0.84)² ≈ 7.85 doubled across two arms. Halving the MDE quadruples n.
- **Randomize, then block what you can't randomize away:** randomization handles confounders in expectation but leaves residual imbalance in finite samples. Block/stratify on strong known prognostic variables (site, device, baseline severity) to reduce variance and guarantee balance on them. Randomize *within* blocks.
- **Choosing the analysis unit:** the unit of randomization must match the unit of analysis and the level at which interference occurs. If treating one user spills over to others (social features, marketplaces, shared inventory), randomize at the cluster level (geo, market, time), not the user — otherwise SUTVA is violated and your estimate is biased.
- **When to stop:** either fix n in advance and analyze once, or use a proper sequential design (group-sequential with alpha-spending, or always-valid/e-value methods) that controls error under repeated looks. Never eyeball a dashboard and stop when it hits significance.
- **Confounder vs collider decision:** include a variable as a control only if it's a common cause of treatment and outcome (back-door path to block). Exclude it if it's a mediator (on the causal path — controlling it removes the very effect you want) or a collider (common effect of two variables — controlling it opens a spurious path).

## Control and randomization detail

- **The counterfactual question.** A control answers "what would these same units have done without treatment?" Choose the control that isolates the one thing you're testing: a placebo controls for expectation effects; a sham controls for the procedure; an active comparator controls for "is it better than current practice." A no-treatment control that differs in attention, procedure, or timing confounds the treatment with those differences.
- **Randomization vs adjustment.** Randomization balances *all* confounders — including ones you never measured or thought of — in expectation. Statistical adjustment only fixes confounders you measured and modeled correctly. Prefer randomization; use adjustment (and blocking) to reduce residual variance, not as a substitute.
- **Blocking/stratification** groups similar units and randomizes within groups, guaranteeing balance on strong prognostic variables and shrinking variance. Block on the few variables most predictive of the outcome (site, device, baseline level); don't over-block into tiny cells.
- **Paired/within-subject designs** remove between-unit variance entirely and can slash the required n, but only when carryover and order effects are absent or counterbalanced.

## Failure modes and pitfalls

- **Peeking / optional stopping.** Repeatedly testing significance as data accrues and stopping at the first p < 0.05 inflates the false-positive rate dramatically (toward ~25–40% with frequent looks instead of 5%). Fix: pre-set n, or use group-sequential boundaries / always-valid inference. Continuous dashboards are peeking machines — gate the decision, not the chart.
- **Post-hoc power theater.** "The result was null but power was low" computed from the observed effect is circular and uninformative. Report the confidence interval instead — it directly shows which effects are ruled out.
- **Multiple comparisons unaccounted.** Testing many metrics/variants/subgroups and celebrating the significant one. With 20 independent metrics at α=0.05, ~1 false positive is expected by chance. Pre-declare one primary metric; correct (Bonferroni for few, Benjamini-Hochberg FDR for many) for the rest, which are secondary/exploratory.
- **Novelty and primacy effects.** Users react to *change* itself; an early lift can be curiosity that decays, or regular users temporarily disrupted. Run long enough to reach steady state; segment new vs returning users; watch the trend, not just the cumulative mean.
- **Interference / SUTVA violations.** In marketplaces, social networks, or shared-resource systems, treating some users affects control users (budget depletion, network effects), biasing the contrast toward zero or the wrong sign. Use cluster/switchback randomization.
- **Controlling for a collider or mediator.** Classic: conditioning on a post-treatment variable. Example — adjusting for "clicked" when estimating an ad's effect on purchase induces selection bias because clicking is a collider/mediator. Only pre-treatment confounders belong in the adjustment set.
- **Simpson's paradox from ignoring a confounder** — an aggregate effect reverses within every subgroup because group membership confounds. Stratify by the confounder; trust the within-stratum (or properly adjusted) estimate, not the pooled one.
- **Chasing statistical over practical significance.** Enormous samples make trivial effects significant. Always ask: is the point estimate big enough to act on? Judge against the pre-set decision threshold.
- **Unbalanced ablations in ML.** Changing two things at once (architecture and learning rate), or comparing at unequal compute/data budgets, so you can't attribute the delta. Ablate one variable at a time, hold compute/data/hyperparameter-tuning budget matched, and report seeds/variance — a single-seed "win" inside run-to-run noise is not a result.
- **Ignoring variance/replication.** Reporting a single run or ignoring the confidence interval. ML results especially need multiple seeds; the seed-to-seed spread often dwarfs the claimed improvement.
- **Baseline drift / non-comparable periods.** Comparing treatment this week to control last week conflates the treatment with time (holidays, releases). Randomize concurrently.
- **Survivorship / attrition bias.** Analyzing only units that completed (didn't churn, didn't drop out) breaks randomization if attrition is outcome-related. Use intention-to-treat.
- **Regression to the mean mistaken for an effect.** Selecting units because they were extreme (worst-performing stores, sickest patients) and then observing improvement — much of which is just regression to the mean, not the intervention. A concurrent randomized control catches this; a pre-post comparison on a selected group does not.
- **Pseudoreplication.** Treating correlated measurements (repeated measures on the same subject, multiple cells from one animal) as independent, inflating the effective n and shrinking p-values spuriously. The independent unit is the unit of randomization; use mixed models or aggregate to that level.
- **Ceiling/floor effects.** If the metric is near its maximum (or minimum) at baseline, a real treatment effect can't register. Check the metric has headroom before designing.
- **Non-inferiority vs superiority confusion.** "No significant difference" does not prove equivalence. A non-inferiority claim requires a pre-specified margin and adequate power to rule out a meaningful decrement — it is a different design, not a failed superiority test.
- **Uncontrolled multiple looks across variants (A/B/n).** Running many variants at once multiplies false positives just like multiple metrics. Correct across variants, or use a design (e.g., multi-armed bandit with proper inference) built for it.
- **Confusing the null with the alternative you care about.** Failing to reject H₀ at your n says the data are consistent with no effect *and* with small effects your study couldn't detect. Report the CI so readers see what's excluded.
- **Optimizing a proxy metric that diverges from the true goal (Goodhart).** A/B-winning on clicks can lose on retention or revenue. Pick a metric that is causally upstream of the real objective, and monitor guardrail metrics for degradation.

## Worked micro-examples

All reproducible cold by a strong model (n≈30k/arm from 16·p(1−p)/Δ²; graduation as mediator+collider → leave out; ~20% false-positive rate at 10 looks; matched-compute multi-seed ablation; marketplace interference → switchback). Two sharpenings worth keeping because the model's *default* phrasing is looser than the correct rule:
- **Simpson's paradox has no purely statistical answer — the DAG decides.** Trust the *stratified* estimate only when the stratifier is a confounder (the usual textbook severity case); trust the *pooled* estimate when the stratifier is a collider or post-treatment mediator. "Always trust the subgroups" is wrong.
- **Sizing scales as 1/Δ²:** halving the effect you'll act on quadruples n. The decision threshold, not the statistics, sets the scale — so pin the decision first.

## Verification and self-check

- **State the decision and threshold first**, then confirm the design can detect the MDE at adequate power — computed before data.
- **Draw the causal graph** and justify each variable in (confounder) or out (mediator/collider) of the adjustment set; never condition on post-treatment variables.
- **Report effect size with a confidence interval** and judge against practical significance, not just p < 0.05.
- **Check the analysis discipline:** one pre-declared primary metric; corrections for multiplicity; fixed n or a valid sequential procedure (no peeking).
- **Check for interference** (does treating one unit affect another?) and randomize at the right cluster level; use intention-to-treat against attrition.
- **For ML:** one variable per ablation, matched compute/data/tuning, multiple seeds, variance reported.
- **Ask what would make this result wrong** — novelty decay, confounded time periods, survivorship — and confirm the design rules each out.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 10 claims: ~10 baseline. Opus 4.8 cold produced the sample-size formula (~31k/arm), the mediator/collider adjustment rule via the backdoor criterion, ~20% peeking inflation, marketplace SUTVA + switchback fix, post-hoc-power-is-circular → report CI/TOST, matched-compute multi-seed ablation with bootstrap testing, and non-inferiority/TOST — and it *exceeded* the skill on Simpson (correctly making it DAG-dependent) and on A/B pitfalls (added SRM and Twyman's law unprompted).
- No knowledge delta. The skill was slightly *behind* Opus on Simpson's paradox; that section was tightened to the DAG-dependent rule so the skill no longer under-teaches relative to the base model.
- Worked examples compressed to the two rule-sharpenings; the arithmetic was reproducible.
