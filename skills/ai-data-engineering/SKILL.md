---
name: ai-data-engineering
description: Load when curating, cleaning, labeling, or formatting datasets for training or fine-tuning AI models — deduplication, decontamination, synthetic data generation, LLM-assisted labeling, chat-format preparation, data versioning, or auditing a dataset before a training run.
---

# Data Engineering for AI Systems

Assumed baseline (verified expert-grade cold): filtering cascade cheap→expensive with decontamination as the final gate on final text; MinHash/LSH at ~128 perms, Jaccard ~0.8, 5-gram word shingles with normalization before shingling; dedup on the prompt for instruction data; dedup/decontaminate the merged pool *before* splitting; 13-gram exact-match decontamination for pretraining with stricter fuzzy matching for fine-tuning; model collapse conditional (replace vs accumulate; verifiers make synthetic near-risk-free); kappa not percent agreement, <0.4 fix the rubric, disagreements are rubric ambiguity; completion-only loss masking and its "model generates User:" symptom; truncation audit with p50/p95/p99 and the completion-cut → trailing-off mechanism; fine-tune-worse-than-base diagnosis starting at eval/template format; datatrove + NeMo Curator (also text-dedup, dolma) as the 2026 production tools; content-hashed lineage with dataset hash in the training config.

## Discipline rules

- **Read 30 fully rendered examples** (template applied, exactly as the trainer sees them) before any run; re-read after fixes; ship when a fresh sample yields no new defect *categories* (individual bad examples <~1% are acceptable; systematic patterns are not).
- **One shared rendering function** imported by trainer and inference stack, plus a byte-diff test in CI: `render(sample) == inference.build_prompt(sample)`. Template drift as subtle as a trailing newline or an `add_generation_prompt` mismatch teaches the model a dialect inference never speaks.
- Decontamination reads an **eval registry**, not a hand-typed benchmark list — internal evals are the ones nobody remembers. Log removals; >2% contamination from one source is a signal about the source.
- Gates before training: completion-cut count = 0; cross-split near-dup rate = 0 at threshold; composition table (source/language/length/label) before vs after filtering — no domain silently vanished.

## Sharpenings and priors the strong baseline lacks

- **Priors for "the fine-tune underperforms"**: ~60% data problem, ~25% formatting/template, ~15% everything else — budget debugging time accordingly, even though tuning is more fun. And the quality anchor: aggressive filtering keeping 10–20% of raw data (Phi/FineWeb-Edu lineage) beats training on everything; a few hundred *bad* examples actively damage 10k good ones — deleting data is usually the highest-ROI edit.
- **Fine-tuning decontamination operating point**: MinHash at ~**0.6** (looser than the 0.8 dedup threshold) against eval items — paraphrased eval questions still leak the answer pattern; exact n-grams miss them.
- **Prompt-dedup asymmetry**: near-identical prompts with different completions teach the model outputs are arbitrary (resolve conflicts explicitly); near-identical *completions* across different prompts are often fine (refusals). Keep the longest/highest-quality cluster representative, not a random one.
- **Epoch asymmetry**: repeating the top quality tier 2–4× is safe and often beneficial; repeating templated/low-quality data amplifies its artifacts. Record mixture weights, epochs-per-source, and shuffle seed in lineage — "the same dataset" with different weights is a different distribution.
- **Capability-regression blend**: a 100%-task fine-tune degrades general instruction following; blend 5–25% general instruction data back and validate the ratio on a general eval — don't guess and ship.
- **Synthetic diversity is engineered, not hoped for**: seed every generation with a distinct real example/persona, mix ≥2 teacher models, and *alert* on diversity metrics (distinct n-gram ratio, embedding dispersion vs the real seed set). 30% of completions opening with the same three words is poisoning the model's entropy. Verify label invariance on paraphrase-augmented classification data (rerun the labeler; drop disagreements — negation/hedging flips labels silently).
- **Read the rejects**: sample 100 *rejected* items per filter stage and report per-stage kill rates — a miscalibrated quality filter silently deletes an entire domain (all code, all non-English). Length filters kill brevity: "drop <200 chars" removes every legitimate short answer and refusal, then the model can't be brief.
- **Judge/labeler from the same family as the generator** inflates synthetic quality via self-preference — cross-family judge plus human audit. Pin the labeler model version; provider updates drift labels silently — re-run the calibration set on any change, keep 1–3% permanent human sampling with kappa tracked.
- **PII scrubbing happens before the data lake**, not before training — otherwise the "clean" set coexists with raw dumps in five buckets; keep a scrub manifest (rule version, what was redacted) in lineage. Dedup itself is a privacy mitigation (memorization scales with duplication count).
- Rubric economics: three rubric iterations on n=100 cost less than one relabel of n=50,000. Human–human kappa is the ceiling; don't demand LLM–human agreement above it.

## Worked audit (the two checks that catch most silent damage)

```python
# 1) Truncation: does max_seq_len cut loss-bearing tokens?
full = tok(render(ex), add_special_tokens=False).input_ids
if len(full) > MAX_SEQ_LEN:
    prefix = tok.apply_chat_template(ex["messages"][:-1], tokenize=False, add_generation_prompt=True)
    if len(tok(prefix, add_special_tokens=False).input_ids) < MAX_SEQ_LEN:
        completion_cut += 1   # answer partially truncated → model learns to trail off; drop or shorten
    # else: prompt alone overflows → drop entirely
# Also inspect the tail: examples >3× median length are usually concatenation bugs, not long documents.

# 2) Template drift: byte-diff train rendering against the live inference path
assert render(sample) == inference_stack.build_prompt(sample), "train/inference template drift"
```

## Verification / self-check

1. 30 rendered examples read; zero new defect categories.
2. Byte-diff passes; EOS present on every completion (or the model rambles past answers).
3. completion_cut = 0; over-length <1% or max_seq_len raised knowingly.
4. Dedup + decontamination reports against the full eval registry, removals logged.
5. Composition table pre/post filtering; labeler kappa ≥ human-human minus noise; dataset hash recorded where training can't run without it.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Biggest baseline gaps found: fine-tuning decontamination's numeric operating point (MinHash ~0.6, looser than dedup); otherwise Opus produced the cascade, MinHash parameters, kappa discipline, truncation audit, and 2026 tooling (datatrove/NeMo Curator) cold.
- Retained value: quantitative priors (60/25/15, keep-10–20%, 5–25% general blend, 2–4× top-tier epochs), diversity alerting for synthetic data, read-the-rejects discipline, and the two-check audit code.
