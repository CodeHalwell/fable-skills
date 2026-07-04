---
name: llm-evaluation
description: Load when designing, reviewing, or debugging evaluations of LLM systems — eval sets, LLM-as-judge setups, regression testing of prompts, A/B comparison of models or prompts, or when someone claims "model/prompt X is better than Y" and the evidence needs scrutiny.
---

# Evaluating LLM Systems

## Core mental model

- **The eval set is the spec.** Build it before (or at worst alongside) the system. If you can't write 30 input/expected-behavior pairs, you don't understand the task well enough to build it, and no amount of prompt iteration will converge. Iterating on a prompt without a fixed eval set is Brownian motion: each "fix" silently regresses cases you're no longer looking at.
- **Task-grounded beats generic.** MMLU/HELM-style benchmark scores tell you almost nothing about whether a model can do *your* task with *your* prompt and *your* data distribution. A 2-point benchmark gap is routinely dominated by a prompt change. Always evaluate the full system (prompt + retrieval + model + parsing), not the model in isolation.
- **Aggregate scores hide failure clusters.** A system that is 92% accurate overall but 40% accurate on the 15% of traffic that is non-English, or on inputs longer than 4k tokens, is a broken system with a good-looking dashboard. Always slice.
- **Every judge is a model with its own failure modes.** LLM-as-judge is a measurement instrument that must itself be calibrated against human labels before you trust it. An uncalibrated judge measures "what the judge likes," which correlates with verbosity, confidence, and style — not correctness.
- **Small n means noisy deltas.** With 100 eval items, a 3-point accuracy difference is usually noise. Treat every comparison as a statistics problem, not a leaderboard.

## Decision framework: choosing the eval method

| Situation | Method | Why |
|---|---|---|
| Output has a checkable ground truth (classification label, extracted field, exact answer, code that runs) | Programmatic assertion (exact match, normalized match, unit tests, regex on structure) | Deterministic, free, zero judge bias. Always prefer when possible — many "generative" tasks hide a checkable core. |
| Output is free-form but has objective criteria (contains required facts, no hallucinated entities, follows format) | LLM judge with a **binary rubric per criterion** | Binary pass/fail per criterion is far more reliable than 1–10 scoring; scores 4–7 on a Likert scale are judge noise. |
| Comparing two systems on subjective quality (tone, helpfulness, writing) | **Pairwise** LLM judge with position swap, then aggregate win rate | Absolute scores drift with judge mood and prompt phrasing; pairwise preferences are much more stable and match how humans actually judge. |
| High-stakes launch decision, safety-relevant behavior, or judge–human agreement unknown | Human eval (see mandatory cases below) | Judges have not earned trust here. |
| Long-form factuality | Decompose into atomic claims, verify each claim (against source docs or search), score = fraction supported | Judging a whole essay "factual? y/n" misses individual fabrications; claim-level checking finds them. |

**Human eval is mandatory, not optional, when:** (1) calibrating a new LLM judge (collect 50–200 human labels, measure judge–human agreement — target agreement at least as high as human–human agreement on the same items); (2) the eval decides a launch or a model migration; (3) the failure mode is subtle (sycophancy, hedging, plausible-but-wrong reasoning) — judges are systematically bad at these; (4) safety/harm assessment.

## Building the eval set

- Sources, in order of value: real production failures > real production traffic (sampled + labeled) > synthetically perturbed real examples > fully synthetic examples. Fully synthetic sets overrepresent cases the generating model finds easy, which are correlated with what the evaluated model finds easy — inflating scores.
- Size guidance: 30–50 items is enough to catch gross regressions; 200–500 for comparing systems within a few points; per-slice minimums matter more than the total (at least ~30 per slice you intend to report).
- Deliberately include: adversarial/edge inputs, inputs where the correct answer is "I can't do that / not enough information" (models overfit to always answering), long inputs, inputs resembling but not matching common patterns (tests memorization vs. reasoning).
- **Freeze it and version it.** Eval set changes must be a diff-reviewed commit, otherwise scores are incomparable across time. Keep a held-out set you look at rarely (see overfitting, below).

## LLM-as-judge design rules

1. **Rubric first.** Write explicit, binary criteria: "Does the answer cite at least one source from the provided context? yes/no." Never ask "rate quality 1–10" without anchored definitions per point — and even then, prefer decomposing into binaries and summing.
2. **Pairwise with position swap.** Present (A, B) and (B, A); count a win only if consistent across both orders, else record a tie. Judges have a measurable position bias (often preferring the first or last response depending on the judge model) — an unswapped pairwise eval can flip its conclusion.
3. **Separate criteria into separate judge calls** when they interact (e.g., correctness and style) — a single call lets a well-written wrong answer bleed into the correctness score (verbosity/eloquence bias).
4. **Give the judge the reference/ground truth when one exists.** A judge grading "is this answer correct?" without the reference is just a second model doing the task, often worse than the system under test.
5. **Judge-model contamination (self-preference):** a judge from the same model family as a candidate systematically favors that candidate's outputs — same style, same idioms, same reasoning patterns. When comparing model A vs model B, don't use A or B (or their siblings) as the sole judge; use a third-family judge, or better, two judges from different families and check they agree. If they disagree materially, that subset goes to humans.
6. **Force reasoning before the verdict**, and make the verdict machine-parseable (e.g., end with `VERDICT: A` / `VERDICT: B` / `VERDICT: TIE`). Parsing free-text judgments introduces its own error rate.
7. **Pin judge temperature to 0 and pin the judge model version.** A judge that changes under you invalidates all longitudinal comparisons — a "regression" after a judge-model update is the most common false alarm in eval dashboards.
8. Re-run the judge on a fixed calibration set whenever anything about the judge changes; alert if agreement with stored human labels drops.

## Statistical significance with small eval sets

- Default tool: **paired bootstrap** on the per-item score differences. Same items evaluated by both systems means paired analysis — never compare two independent confidence intervals (that throws away the pairing and hugely overstates uncertainty).
- Quick significance check for paired binary outcomes: **McNemar's test.** Only the discordant pairs matter — items where A is right & B wrong (call it `b`) vs A wrong & B right (`c`). If `b + c` is small (say < 25), the systems are statistically indistinguishable regardless of the headline accuracy gap.
- Rule-of-thumb for binary accuracy: the 95% CI half-width is roughly `1/sqrt(n)`. n=100 → ±10 points; n=400 → ±5; n=2500 → ±2. Internalize this: it kills most "we improved 2%" claims on n=150 instantly.
- Nondeterminism: even at temperature 0, LLM outputs vary across runs (batching, hardware). For close comparisons, run each system 3–5 times per item and use the per-item mean; report variance across runs so readers can see the noise floor.
- Multiple comparisons: if you slice results 10 ways, expect one slice to look "significantly" different by chance. Flag exploratory slices as hypotheses, confirm on fresh data.

## Regression testing prompts

- Treat prompts like code: every prompt change runs the eval suite in CI before merge. Store prompt + model version + eval score together.
- Two suite tiers: a fast **smoke suite** (20–50 items, programmatic checks only, runs on every change) and the full suite (judge-based, runs pre-release or nightly).
- Assert on **behaviors, not exact strings**: "output parses as JSON," "contains no URLs not present in context," "refuses this category of request," "mentions the required disclaimer." Exact-string assertions break on harmless rephrasing and train people to ignore red CI.
- Every production incident becomes a permanent eval case (same discipline as adding a regression test for a bug).
- When migrating model versions, run the full suite on both and diff **per-item**, not aggregate — an equal aggregate score can hide 10% of items flipping right→wrong and another 10% flipping wrong→right, which is a large behavioral change users will feel.

## Failure modes & pitfalls

- **Iterating on the test set.** You tweak the prompt until the eval passes; after 30 iterations the prompt is overfit to those exact items and production quality hasn't moved. Correction: dev/held-out split even for prompt engineering; touch the held-out set only at decision points, and refresh it periodically from production.
- **Eval-set contamination.** Items copied from public benchmarks or popular datasets are in the model's training data; the model has memorized answers. Symptom: suspiciously perfect performance that collapses on paraphrases. Correction: build from private/production data; spot-check by paraphrasing 20 items — a big score drop indicates memorization.
- **Judging the judge never.** Teams ship judge-based dashboards without ever measuring judge–human agreement. Correction: no judge in the reporting path without a calibration measurement; report the judge's error rate alongside its scores.
- **Likert-scale scoring.** "Rate 1–10" outputs cluster at 7–8 regardless of quality, differ across judge model versions, and are incomparable across rubrics. Correction: binary criteria or pairwise.
- **Verbosity bias.** Pairwise judges prefer longer answers at a measurable rate even when the longer answer is worse. Correction: instruct the judge explicitly that length is not quality; check win rate vs. length-difference correlation in your results — if win probability rises monotonically with token count, your judge is measuring length.
- **Comparing across different eval sets or judge versions.** "Last quarter we scored 78, now 84" is meaningless if the set or judge changed. Correction: version both; recompute old systems on the new set when the set changes.
- **Ignoring refusals/format failures in the metric.** A response that fails to parse or refuses is often silently dropped, inflating accuracy. Correction: unparseable/refused = wrong, and track parse-failure rate as its own metric.
- **Reporting only the mean.** Correction: always report n, CI, and the 2–3 worst slices. A results table without n is an anecdote.
- **Testing at a different temperature/config than production.** Eval at temp 0, serve at temp 1 → eval measures a different system. Match serving config exactly, including system prompt and max_tokens (truncation causes real failures).
- **Cluster-blind sampling.** 500 eval items where 400 are near-duplicates of one input pattern is effectively n≈100 with a misleading label. Deduplicate/stratify before trusting n.

## Worked micro-example: is prompt B actually better?

Prompt A: 78/100 correct. Prompt B: 84/100. Ship B?

```python
# Paired comparison — same 100 items for both.
# Discordant counts: A right & B wrong: b = 6;  A wrong & B right: c = 12.
from scipy.stats import binomtest
p = binomtest(6, 6 + 12, 0.5).pvalue   # exact McNemar
print(p)  # ≈ 0.24 — not significant
```

Only 18 items distinguish the prompts, and 6 vs 12 splits are common under pure chance (p ≈ 0.24). Rule-of-thumb check agrees: at n=100 the noise floor is ~±10 points and the gap is 6. Correct action: don't conclude B is better yet — expand the eval set (targeting the discordant item types, which show you *where* the prompts differ), rerun, and require the gap to survive. Also read all 18 discordant items by hand: at this scale, reading beats statistics for deciding what to fix next.

## Verification checklist before presenting eval results

- [ ] n stated for every number; paired test or bootstrap CI on every A-vs-B claim.
- [ ] Judge (if any) calibrated against human labels; agreement number reported; judge model/version pinned; not from the same family as any candidate (or cross-family agreement checked).
- [ ] Pairwise judgments position-swapped; win-rate vs. length correlation checked.
- [ ] Results sliced by the 3–5 dimensions most likely to hide clusters (language, input length, category, difficulty); no reported slice under n≈30.
- [ ] Parse failures and refusals counted as failures, reported separately.
- [ ] Eval config identical to production (model, temperature, system prompt, max_tokens).
- [ ] Eval set is version-pinned, not contaminated by training data or by your own prompt iteration.
- [ ] You personally read at least 20 raw transcripts, including failures. If the metric and your reading disagree, the metric is wrong until proven otherwise.
