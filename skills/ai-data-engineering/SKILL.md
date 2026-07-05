---
name: ai-data-engineering
description: Load when curating, cleaning, labeling, or formatting datasets for training or fine-tuning AI models — deduplication, decontamination, synthetic data generation, LLM-assisted labeling, chat-format preparation, data versioning, or auditing a dataset before a training run.
---

# Data Engineering for AI Systems

## Core mental model

- **The dataset is the model.** Architecture and hyperparameters are commodity; the data distribution is the product. An hour spent reading 200 random training examples reliably beats an hour of hyperparameter search. If a fine-tune underperforms, the prior is roughly: 60% data problem, 25% formatting/template problem, 15% everything else. Check data first even though tuning is more fun.
- **Quality beats quantity at every scale.** Phi-style "textbook quality" results and the FineWeb-Edu lineage established that aggressive filtering (keeping 10–20% of raw data) outperforms training on everything. For fine-tuning the effect is stronger: 1,000 excellent examples beat 50,000 mediocre ones, and a few hundred *bad* examples (wrong labels, truncated answers, format drift) actively damage a model trained on 10,000 good ones. Deleting data is usually the highest-ROI edit.
- **The filtering cascade has an order for a reason:** cheap exact dedup → heuristic quality filters (length, language ID, boilerplate/repetition ratios) → near-dedup (MinHash/LSH) → model-based quality scoring → decontamination against every eval you will ever report. Run cheap filters first so expensive ones see less data; run decontamination *last* so nothing re-introduces contaminated items after the check.
- **Contamination is the silent evaluation killer.** Train/test leakage doesn't crash anything — it just makes every number you report a lie, and you find out when production disagrees with your eval. Treat decontamination as a release gate, not hygiene.
- **Reproducibility is lineage, not luck.** Every training set must be reconstructible: raw source + versioned filter code + seed = exact dataset. If you can't answer "which examples were in the run that produced model v12," you cannot debug model v12.

## Deduplication engineering

Two distinct problems; don't conflate them:

1. **Exact/near-exact dedup**: hash normalized documents (MD5/SHA of lowercased, whitespace-collapsed text). Cheap, always do it, catches mirror pages and re-scrapes.
2. **Near-duplicate dedup**: MinHash + LSH over character or word shingles. Standard operating point (what NeMo Curator ships as defaults, as of 2026): ~128 permutations, Jaccard threshold ≈ 0.8, 5-gram word shingles. Below ~0.7 you start deleting legitimate paraphrases; above ~0.9 you miss templated spam that differs only in inserted entities.

Reasoning chain for choosing the setup:
- *How big is the corpus?* < 1M docs: `datasketch` MinHashLSH in one process is fine. 10M–1B: datatrove (Hugging Face's FineWeb pipeline library) or NeMo Curator (GPU-accelerated MinHash) — as of 2026 these are the two production-standard open tools.
- *What's the duplication structure?* Web scrapes: dedup at document level. Instruction data: dedup on the *prompt* (or prompt prefix), not the full example — near-identical prompts with different completions teach the model that outputs are arbitrary; near-identical completions across different prompts are often fine (e.g., refusals).
- *Cluster resolution:* LSH gives you connected components of near-dups. Keep one representative per component — pick the longest or highest-quality-score member, not a random one.
- **Order matters:** dedup the *combined* pool before splitting train/val/test. Splitting first and deduping within splits leaves near-duplicates straddling the boundary — the most common cause of "our val loss is amazing, production is not."

**Decontamination** is dedup with the eval set as the query side: n-gram overlap (13-gram substring match is the classic operating point for pretraining; for fine-tuning use MinHash at a *lower* threshold, ~0.6, because paraphrased eval questions still leak the answer pattern). Decontaminate against every benchmark you report, plus your internal eval sets — internal evals are the ones nobody remembers to check. Log what was removed; a 5% contamination rate against a benchmark you didn't build from that source is itself a signal your data vendor included it.

## Synthetic data judgment

The 2026 consensus, distilled: model collapse is real but conditional. Error compounds when synthetic data *replaces* real data across generations; *accumulating* synthetic alongside a persistent real seed is stable. Judgment framework — ask in order:

1. **What gap am I filling?** Synthetic works when *targeted*: underrepresented classes, hard negatives near a decision boundary, rephrasings/format augmentation of real seeds, long-tail languages or edge cases you can specify but can't collect. It fails as a *substitute for coverage you don't understand* — a model generating "diverse user queries" from nothing produces the generator's idea of diversity: low-entropy, mode-seeking, systematically missing what real users actually do.
2. **Is there a verifier?** Synthetic data with programmatic verification (code that must pass tests, math with checkable answers, extraction against source docs) is nearly risk-free — generate 10×, filter to the verified subset. Unverifiable synthetic (open-ended chat quality) needs LLM-judge filtering plus human audit, and the judge shares blind spots with the generator.
3. **Diversity controls in the pipeline, not hope:** seed every generation with a distinct real example or persona/attribute combination; mix at least two teacher models; measure output diversity (distinct n-gram ratio, embedding-space dispersion vs. the real seed set) and alert when it drops. A synthetic set where 30% of completions start with the same three words is poisoning your model's entropy.
4. **Ratio discipline:** keep real data in every mix; for general-capability fine-tuning stay minority-synthetic unless verified. Targeted, verified synthetic can be 100% of a *slice* safely.

## Data mixing and curriculum (when assembling from multiple sources)

- Mixture weights matter more than most hyperparameters. Reason about them explicitly: start from natural proportions, then upweight scarce-but-critical domains (code, math, your product's domain) rather than downweighting everything else — and record the weights in lineage.
- Epoch asymmetry: repeating high-quality data 2–4× is generally safe and often beneficial; repeating low-quality or templated data amplifies its artifacts. If you must repeat, repeat the top quality tier only.
- For fine-tuning mixes, guard against **capability regression**: a narrow fine-tune on 100% task data degrades general instruction following. Standard mitigation is blending 5–25% general instruction data back in; validate the ratio on a general-capability eval, don't guess and ship.
- When two sources cover the same domain, dedup *across* sources before weighting — otherwise your "40/60 mix" is silently a different ratio after overlap.

## Labeling economics (LLM-as-labeler)

- Default 2026 architecture: LLM labels everything cheaply; humans audit a stratified random sample (oversample low-confidence and rare classes); disagreements drive rubric iteration. Human-labels-everything is justified only for safety-critical gold sets and judge calibration.
- Measure agreement with **Cohen's/Fleiss' kappa or Krippendorff's alpha, never raw percent agreement** — 90% raw agreement on a 95/5 class split is worse than chance-adjusted useless. Kappa < 0.4: your rubric is ambiguous, fix the rubric before scaling. Kappa 0.6–0.8: usable with audit. Human–human kappa is the ceiling; don't demand LLM–human agreement above it.
- Rubric iteration loop: label 100 → find disagreement clusters → they are almost always *rubric ambiguity*, not labeler error → rewrite the rubric with explicit tie-breakers and 3–5 worked borderline examples → relabel the same 100 → repeat until kappa stabilizes. Only then run the full set. Budget-wise, three rubric iterations on n=100 cost less than one relabel of n=50,000.
- Keep the audit permanent: 1–3% ongoing human sampling with kappa tracked over time. LLM labeler drift (provider model updates) is real and silent — pin the labeler model version and re-run the calibration set when it changes.

## Fine-tuning data formatting

- **The chat-format mismatch trap** (single most common fine-tuning bug): the model was pretrained/instruction-tuned with a specific chat template (special tokens, role markers, EOS placement). If your training examples are rendered with a different template — or you concatenate `"User: ... Assistant: ..."` strings manually while inference uses the real template — you're teaching the model in a dialect it never speaks at inference. Always render with the tokenizer's own template (`tokenizer.apply_chat_template(...)` in Hugging Face) and verify by decoding a tokenized training example and comparing byte-for-byte against what inference sends.
- Mask the loss correctly: compute loss on assistant tokens only (completion-only masking) unless you deliberately want the model to learn to imitate users. Check by decoding the unmasked token spans of one batch.
- Template *consistency*: one system-prompt convention across the set. If half your examples embed instructions in the system slot and half in the user turn, you dilute both.
- EOS discipline: every completion must end with the EOS the inference stack stops on, or your fine-tuned model will ramble past its answer.

## Tokenizer-aware analysis (the truncation audit)

Before any run, compute the token-length distribution of *fully rendered* examples (template included) with the *training* tokenizer. Report p50/p95/p99/max against `max_seq_len`. Then check *what* gets truncated: if truncation cuts completions (the part you compute loss on), those examples teach the model to produce cut-off answers — drop or shorten them instead. A dataset where 8% of examples silently lose their final answer to truncation reliably produces a model that trails off. Also inspect the tail: examples > 3× median length are often concatenation bugs or scraped junk, not legitimate long documents.

## Privacy: PII in training data

- Regex alone catches formatted PII (emails, phones, SSNs, credit cards — use `presidio` or equivalent, not hand-rolled regex); it misses names, addresses, and free-text identifiers — layer an NER pass for those. Accept that recall < 100% and design accordingly: scrub *before* the data lake, not before training, so no unscrubbed copy persists.
- Dedup helps privacy: memorization risk scales with duplication count, so near-dedup is itself a mitigation.
- Keep a scrub manifest (what was redacted, by which rule version) in lineage — you will be asked.

## Versioning and lineage

- Version the *recipe*, snapshot the *result*: raw data is immutable and content-addressed; every derived set is (raw refs + filter code commit + config + seed), plus a materialized snapshot with a hash. Tools: Hugging Face datasets revisions, DVC, LakeFS, or plain object-store prefixes with a manifest — the discipline matters more than the tool.
- Every trained model's metadata must record the dataset hash. "Retrain last month's model" must be a lookup, not archaeology.

## How an expert thinks through it: "the fine-tune got worse"

A team fine-tunes on 40k support conversations; the new model scores lower than the base model on their eval. Internal monologue:

*Base model beats its own fine-tune — that's not underfitting, that's the training data actively teaching something wrong. Before touching LR or epochs (rejected: hyperparameters rarely flip the sign of a fine-tune), read 30 random rendered examples exactly as the trainer sees them.* → Finds the examples were rendered with a generic `### Instruction:` template, but inference uses the model's chat template. *That alone can explain it. But keep looking — bugs cluster.* → Token audit: p99 is 6k tokens vs. max_seq_len 4k; 11% of examples truncate mid-answer. *Second real problem.* → Check split hygiene: eval set was sampled from the same ticket pool after splitting by *conversation ID*… but the same customer issue spawns near-identical tickets. MinHash across train/eval at 0.8: 7% of eval has a near-dup in train. *So the previous "improvement" numbers were inflated too — the baseline comparison is polluted in both directions.* Fix order: re-render with `apply_chat_template`, drop/shorten truncating examples, re-dedup the pool before re-splitting, re-run. *Rejected along the way:* generating synthetic data to "dilute" the bad examples (treats the symptom; the bad examples are still in the gradient), and doubling the dataset (more of a miscooked distribution is more poison). Stopping rule: when a re-read of 30 fresh rendered examples turns up zero surprises and the truncation rate on loss-bearing tokens is ~0, the data is ready; further polishing is procrastination.

## Failure modes & pitfalls

- **Dedup after splitting** (or never re-deduping after merging a new data source into an existing split). Correction: dedup and decontaminate the merged pool, then re-split; treat "add data" as "rebuild dataset."
- **Decontaminating only the benchmarks you remembered.** Internal evals, vendor-supplied test sets, and examples pasted into the eval from the same scrape all leak. Correction: maintain a registry of every eval set; the decontamination job reads the registry, not a hand-typed list.
- **MinHash on raw text without normalization** — case, punctuation, and whitespace differences defeat shingle overlap. Normalize (lowercase, strip punctuation, collapse whitespace) before shingling; keep the original text for training.
- **Prompt-level duplicates with conflicting completions** in instruction data (two labelers, two answers). The model learns high-entropy outputs for that prompt family. Correction: dedup on prompt, resolve conflicts explicitly (keep the better one or rewrite the rubric).
- **`apply_chat_template` at train time but a hand-built prompt string at inference** (or vice versa) — including subtle drift like a missing trailing newline or a `add_generation_prompt=False` mismatch. Correction: one shared rendering function imported by both paths; byte-diff test in CI.
- **Loss on the full sequence for chat fine-tuning**, teaching the model to generate user turns and system prompts. Symptom: model outputs "User:" mid-response. Correction: completion-only loss masking; verify masks by decoding.
- **Judge/labeler grading its own generator's outputs** (same model family) inflates synthetic-data quality scores via self-preference. Correction: cross-family judge, plus human audit on a sample.
- **Percent agreement instead of kappa** on imbalanced labels; teams ship a 92%-agreement labeler that is chance-level on the minority class that matters. Correction: kappa per class, confusion matrix on the human-audited sample.
- **Synthetic rephrasing that drifts labels**: paraphrasing a classification example can silently change the correct label (negation, hedging). Correction: verify label invariance — run the original labeler on the paraphrase and drop disagreements.
- **PII scrubbing after the copy proliferated** — the "clean" dataset coexists with raw dumps in five buckets. Correction: scrub at ingestion; delete raw or lock it to a break-glass path.
- **Filtering on a quality classifier trained on the same distribution you're filtering** without spot-checking the rejects: a miscalibrated filter can silently delete an entire domain (e.g., all code, all non-English). Correction: always sample and read 100 *rejected* items per filter stage; report per-stage kill rates and per-domain composition before/after.
- **"We'll version the data later."** Later is after the model that worked can't be reproduced. Correction: dataset hash in the training config from day one.
- **Length-filtering as a quality proxy without checking what it kills.** "Drop everything under 200 characters" also drops every legitimate short answer, yes/no case, and refusal — then the model can't be brief. Correction: filter on quality signals (repetition ratio, language ID confidence, boilerplate markers), use length only as a weak feature; inspect the length distribution of *kept* data against the target task's real distribution.
- **Mixing epochs and mixture weights unrecorded** — the run is "the same dataset" but a different effective distribution. Correction: the lineage manifest records weights, epochs per source, and the shuffle seed, not just source hashes.

## Worked micro-example: near-dedup + decontamination gate

```python
from datasketch import MinHash, MinHashLSH
import re

def shingles(text, n=5):
    toks = re.sub(r"[^\w\s]", " ", text.lower()).split()
    return {" ".join(toks[i:i+n]) for i in range(max(1, len(toks)-n+1))}

def minhash(text, num_perm=128):
    m = MinHash(num_perm=num_perm)
    for s in shingles(text):
        m.update(s.encode())
    return m

# 1) Near-dedup the merged pool at Jaccard ~0.8
lsh = MinHashLSH(threshold=0.8, num_perm=128)
keep = []
for i, ex in enumerate(pool):
    m = minhash(ex["prompt"])            # dedup on PROMPT for instruction data
    if not lsh.query(m):
        lsh.insert(f"ex{i}", m)
        keep.append(ex)

# 2) Decontaminate against ALL registered evals at a looser 0.6
decon = MinHashLSH(threshold=0.6, num_perm=128)
for j, ev in enumerate(all_eval_items):     # from the eval registry, not a hand list
    decon.insert(f"ev{j}", minhash(ev["input"]))
clean = [ex for ex in keep if not decon.query(minhash(ex["prompt"]))]
print(f"pool={len(pool)} deduped={len(keep)} clean={len(clean)}")
# Gate: if (len(keep)-len(clean))/len(keep) > 0.02, investigate the source before training.
```

At 10M+ documents, replace this with datatrove's Minhash stages or NeMo Curator's GPU dedup (same parameters, distributed execution) — the logic transfers unchanged.

## Worked micro-example: the truncation + template audit

```python
from transformers import AutoTokenizer
import numpy as np

tok = AutoTokenizer.from_pretrained(MODEL_ID)

def render(ex):
    # THE canonical rendering — same function the trainer and the eval harness import.
    return tok.apply_chat_template(ex["messages"], tokenize=False, add_generation_prompt=False)

lengths, completion_cut = [], 0
for ex in dataset:
    full_ids = tok(render(ex), add_special_tokens=False).input_ids
    lengths.append(len(full_ids))
    if len(full_ids) > MAX_SEQ_LEN:
        # Does truncation eat loss-bearing (assistant) tokens?
        prefix = tok.apply_chat_template(ex["messages"][:-1], tokenize=False,
                                         add_generation_prompt=True)
        if len(tok(prefix, add_special_tokens=False).input_ids) < MAX_SEQ_LEN:
            completion_cut += 1          # answer partially truncated → teaches trailing off
        # else: even the prompt doesn't fit → drop the example entirely

L = np.array(lengths)
print(f"p50={np.percentile(L,50):.0f} p95={np.percentile(L,95):.0f} "
      f"p99={np.percentile(L,99):.0f} max={L.max()} over={np.mean(L>MAX_SEQ_LEN):.1%} "
      f"completion_cut={completion_cut}")
# Gates: completion_cut == 0 after fixes; over-length rate < 1% or raise max_seq_len knowingly.

# Template byte-diff against inference (catches the chat-format mismatch trap):
assert render(sample) == inference_stack.build_prompt(sample), "train/inference template drift"
```

## Verification / self-check

Before declaring a dataset training-ready:
1. **Read 30 fully rendered examples** (template applied, as the trainer sees them). Zero surprises = pass.
2. **Byte-diff** one rendered training example against the exact inference-time prompt for the same input.
3. Truncation audit: fraction of examples losing loss-bearing tokens ≈ 0.
4. Dedup report: near-dup rate across train/val/test = 0 at your threshold; decontamination report against the full eval registry, with removed items logged.
5. Composition table (per source, language, length bucket, label) before vs. after filtering — no domain silently vanished.
6. Labeler kappa on the human-audited sample ≥ your rubric's human–human kappa minus noise.
7. Dataset hash recorded where the training job can't run without it.

Stopping rule: when a fresh random sample read-through produces no new defect *categories* (individual bad examples at < ~1% are acceptable; systematic patterns are not), ship it. Chasing per-example perfection past that point costs more than the marginal model improvement.
