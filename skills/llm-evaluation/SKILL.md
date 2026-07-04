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

## Metric choice details

- Prefer metrics with a decision attached: "answer contains the correct entity" (actionable) over BLEU/ROUGE (uninterpretable for most modern LLM tasks — n-gram overlap punishes valid paraphrase and rewards parroting; use them only for tightly-templated outputs, if at all).
- For classification-shaped tasks, report precision/recall per class, not accuracy — LLM classifiers are often wildly asymmetric (high recall, low precision on the "interesting" class) and the asymmetry is what determines product impact.
- For retrieval-augmented systems, evaluate retrieval and generation *separately* before end-to-end: recall@k of the retriever against labeled relevant docs, then generation quality given gold context. An end-to-end-only eval can't tell you which component to fix, and generation-given-gold-context is the ceiling that tells you whether retrieval is the bottleneck.
- Latency and cost are eval metrics, not afterthoughts: report tokens in/out and p95 latency next to quality. A 1-point quality win at 3x cost is usually a loss; making this visible in the same table changes decisions.
- Calibrate any threshold on dev data, report on held-out. Choosing the judge-score cutoff that maximizes held-out agreement is itself overfitting.

## Eval infrastructure that pays for itself

- Log every eval run as structured records: `{run_id, system_config_hash, eval_set_version, item_id, output, scores, judge_version, timestamp}`. Per-item records are what enable paired statistics, discordant-item review, and longitudinal diffs; aggregate-only logging destroys all three.
- Make single-item replay trivial: `run_eval --item 42 --config prod.yaml` reproducing one failure exactly (same prompt, same context, same params) is the difference between a 5-minute and a 2-hour debugging loop.
- Cache model outputs keyed on (model, params, prompt) — reruns for statistics or new judge versions become free, and you can re-score old outputs with a new rubric without re-generating.
- Keep a "reading queue": every eval run samples 10 random transcripts and 10 failures into a doc a human actually opens. Metrics drift away from reality without this ritual; the discipline of reading transcripts weekly is worth more than another automated metric.

## Regression testing prompts

- Treat prompts like code: every prompt change runs the eval suite in CI before merge. Store prompt + model version + eval score together.
- Two suite tiers: a fast **smoke suite** (20–50 items, programmatic checks only, runs on every change) and the full suite (judge-based, runs pre-release or nightly).
- Assert on **behaviors, not exact strings**: "output parses as JSON," "contains no URLs not present in context," "refuses this category of request," "mentions the required disclaimer." Exact-string assertions break on harmless rephrasing and train people to ignore red CI.
- Every production incident becomes a permanent eval case (same discipline as adding a regression test for a bug).
- When migrating model versions, run the full suite on both and diff **per-item**, not aggregate — an equal aggregate score can hide 10% of items flipping right→wrong and another 10% flipping wrong→right, which is a large behavioral change users will feel.

## Online evaluation: the offline eval is not the end

- Offline eval passing is necessary, never sufficient — production inputs will be weirder than your eval set. Instrument production: sample 1–5% of live traffic, run the calibrated judge on it asynchronously, dashboard the score by day and by slice. This catches regressions your eval set doesn't cover (new input types, upstream prompt-assembly bugs, provider-side model updates).
- Cheap high-signal production metrics that need no judge: parse-failure rate, refusal rate, output length distribution, latency, and user behavioral signals (retry rate, copy rate, thumbs, edit distance between draft and what the user actually kept). A jump in user retries is often the first detectable symptom of a quality regression.
- Implicit signals beat explicit feedback in volume and honesty: thumbs-up/down response rates run ~1% and skew extreme; "did the user accept/edit/abandon the output" covers every interaction.
- Route online failures back offline: the sampled-and-judged production failures are precisely the items to add to next quarter's eval set. This loop — production failure → eval case → fixed → regression-guarded — is the whole game; teams that don't close it re-fix the same failures forever.
- A/B tests are the ground truth for "better": when an offline eval says +5 points and you can afford an experiment, run it, and record the (offline delta, online delta) pair — over time this history tells you how much to trust the offline eval (see ml-production-systems on offline-online correlation).

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

## Worked micro-example: calibrating an LLM judge

You want a judge for "answer is fully supported by the provided context" on a RAG system.

1. Sample 150 production (question, context, answer) triples. Two humans label each `supported` / `unsupported` / `partial` independently. Human–human agreement: they agree on 132/150 (88%). This is your ceiling — no judge can be validated beyond it, and the 18 disagreements define the genuinely ambiguous region.
2. Write the judge prompt: context + answer, instruction to list each factual claim in the answer and mark it supported/unsupported by the context, then output `VERDICT: SUPPORTED` only if all claims pass. Temperature 0, pinned model version, third-family model (not the generator).
3. Run judge on the 132 human-consensus items. Judge agrees on 121/132 (92% of consensus items — above the 88% human-human rate, acceptable).
4. Inspect the 11 disagreements: 7 are the judge marking "supported" for claims that require multi-hop inference from context (judge too lenient on inference), 4 are the judge penalizing correct paraphrase. Add two rubric lines addressing exactly these; re-run; 127/132.
5. Freeze: judge prompt + model version + the 150-item calibration set become a versioned artifact. Every future judge change reruns step 3 automatically; agreement below 90% blocks the change.

Now — and only now — the judge's production numbers mean something, and you can state their error bars: a reported 85% support rate carries roughly ±4% judge error on top of sampling error.

## Verification checklist before presenting eval results

- [ ] n stated for every number; paired test or bootstrap CI on every A-vs-B claim.
- [ ] Judge (if any) calibrated against human labels; agreement number reported; judge model/version pinned; not from the same family as any candidate (or cross-family agreement checked).
- [ ] Pairwise judgments position-swapped; win-rate vs. length correlation checked.
- [ ] Results sliced by the 3–5 dimensions most likely to hide clusters (language, input length, category, difficulty); no reported slice under n≈30.
- [ ] Parse failures and refusals counted as failures, reported separately.
- [ ] Eval config identical to production (model, temperature, system prompt, max_tokens).
- [ ] Eval set is version-pinned, not contaminated by training data or by your own prompt iteration.
- [ ] You personally read at least 20 raw transcripts, including failures. If the metric and your reading disagree, the metric is wrong until proven otherwise.
