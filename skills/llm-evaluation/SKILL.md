---
name: llm-evaluation
description: Load when designing, reviewing, or debugging evaluations of LLM systems — eval sets, LLM-as-judge setups, regression testing of prompts, A/B comparison of models or prompts, or when someone claims "model/prompt X is better than Y" and the evidence needs scrutiny.
---

# Evaluating LLM Systems

## Core mental model (anchors — the discipline is known; the failure is not applying it)

- The eval set is the spec: build it before the system; iterate prompts only against a fixed set (dev/held-out split, production failures > sampled traffic > perturbed-real > fully synthetic).
- Aggregate scores hide failure clusters — always slice; ≥~30 items per reported slice.
- Every judge is a model with failure modes: calibrate against human labels before trusting, pin judge model+temp, binary rubrics over Likert (1–10 clusters at 7–8), pairwise with position swap, third-family judge for A-vs-B, reasoning before machine-parseable verdict.
- Small n means noisy deltas: paired analysis (McNemar/paired bootstrap) on the same items; CI half-width ≈ 1/√n (n=100 → ±10pts); unpaired comparison of two CIs throws away the pairing and overstates uncertainty.
- Even temp-0 outputs vary across runs (batching/floating point); for close calls run 3–5× per item.
- Parse failures and refusals count as wrong and get tracked separately; eval config must match serving config exactly.

## The corrections (what even careful evaluators miss)

**A judge without the reference is just a second model doing the task.** The reflex is to prompt a strong model "is this answer correct?" — that measures the judge's own task ability, often below the system under test. Whenever ground truth or source context exists, put it in the judge prompt and reduce the judge's job to *comparison/entailment*, which is much easier than solving. Corollary for long-form factuality: decompose into atomic claims and verify each against the source — whole-essay "factual? y/n" misses individual fabrications.

**Generation-given-gold-context is the ceiling measurement.** For any retrieval-augmented system, run generation with hand-picked correct context. That number tells you whether retrieval is the bottleneck (gold-context score high, end-to-end low) or the generator is (gold-context score low — no retrieval work will help). Teams evaluate end-to-end only and tune the wrong component for weeks.

**Keep one metric the prompt author cannot see.** Once a judge score is the optimization target, prompts evolve to please the judge (longer, more confident, rubric-keyword-stuffed) — score rises, users feel nothing. Defenses: rotate fresh human calibration samples; alert on score-up-but-user-signals-flat divergence; hold out a metric (or eval slice) from whoever iterates the prompt. Also check win-rate vs length-difference correlation in every pairwise result — a monotonic rise means the judge is measuring length.

**Eval instrumentation leaks into serving.** Rubrics or expected answers end up visible to the system under test via shared config files or copy-paste — producing perfect scores and zero information. Keep eval configs physically separate from serving configs; a suspiciously clean run is a leak until proven otherwise.

**Agent evals: attribute the first wrong step, and fixture the environment.** Aggregate success rate isn't a roadmap; label each failed trajectory's *first* wrong step (wrong tool, wrong args, misread result, gave up, hallucinated a result instead of calling) and let the histogram direct the fixes. Live-API agent evals are flaky by construction: record/replay tool results for regression, reserve live runs for periodic validation. Report per-attempt reliability (and pass^k for "all k succeed"), not pass@k, which flatters unreliable systems.

**Close the production loop or re-fix the same failures forever.** Sample 1–5% of live traffic through the calibrated judge asynchronously; the cheap no-judge signals (parse-failure rate, refusal rate, output-length distribution, user retry/edit/abandon) usually move first. Every sampled production failure becomes a permanent eval case; every incident becomes a regression item. Record (offline delta, online delta) pairs across launches — that history is the only evidence your offline eval can rank candidates at all.

## Compressed checklist (kept for completeness)

- Human eval is mandatory for: judge calibration (50–200 labels; judge–human agreement ≥ human–human agreement, which is the ceiling), launch/migration decisions, subtle failure modes (sycophancy, plausible-but-wrong reasoning), safety.
- Model migration: diff per-item, never aggregate — equal totals can hide 10% right→wrong + 10% wrong→right churn users will feel.
- Two suite tiers: programmatic smoke suite on every change; full judge suite nightly/pre-release. Assert behaviors, not exact strings.
- Include unanswerable/refusal-correct items; dedupe near-duplicate items (400 clones of one pattern is n≈100 wearing a costume); paraphrase-probe for benchmark contamination.
- Log per-item records ({run, config hash, set version, item, output, scores, judge version}); cache outputs keyed on (model, params, prompt) so re-judging is free; single-item replay must be one command.
- Read 10 random + 10 failing transcripts every run; when metric and reading disagree, the metric is wrong until proven otherwise.
- Latency and tokens in/out sit in the same results table as quality; a 1-point win at 3× cost is usually a loss.

## Worked micro-example — the 78 vs 84 verdict

Same 100 items; discordant pairs b=6 (A right, B wrong), c=12. Exact McNemar: `binomtest(6, 18, 0.5).pvalue ≈ 0.24` — not significant; the ±10pt noise floor at n=100 agrees. Correct action: read all 18 discordant items (at this scale reading beats statistics for deciding what to fix), expand the set targeting the discordant item *types*, and require the gap to survive. Ship-B-anyway is defensible only if B is no worse on critical slices and cheaper.

## Verification checklist before presenting results

- [ ] n and paired test/CI on every A-vs-B claim; slices with per-slice n.
- [ ] Judge calibrated (agreement number reported), pinned, reference-supplied where ground truth exists, not same-family as any candidate.
- [ ] Position-swapped pairwise; length-correlation checked.
- [ ] Parse failures/refusals counted as failures; config matches production.
- [ ] Gold-context ceiling measured for retrieval systems; first-wrong-step histogram for agents.
- [ ] One metric or slice invisible to the prompt author; eval configs physically separate from serving.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 14 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus designs judges well but omits supplying the reference/ground truth — its judge is a second model doing the task.
  - No gold-context ceiling measurement for locating the retrieval-vs-generation bottleneck.
  - Goodhart defenses (metric hidden from prompt author, score-vs-user-signal divergence) and eval-instrumentation leakage absent; agent first-wrong-step attribution and record/replay fixtures not surfaced.
