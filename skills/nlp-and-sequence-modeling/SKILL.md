---
name: nlp-and-sequence-modeling
description: Load for NLP and language-model engineering — diagnosing tokenizer-caused model failures, BPE vs unigram choices, picking decoding strategies (greedy/beam/nucleus/temperature) per task, interpreting perplexity correctly, embedding-similarity pitfalls, sequence labeling and span extraction, multilingual/cross-lingual work, and long-sequence handling.
---

# NLP and Sequence Modeling

Assumed baseline (verified expert-grade cold): tokenize-first diagnosis of arithmetic/spelling/whitespace failures; trailing-space token-boundary mechanics; BPE vs unigram and subword regularization (unigram+SentencePiece for morphologically rich low-resource); decoding-per-task table (greedy for closed-form, beam 4–6 + length norm for MT, nucleus 0.9/T 0.7–0.9 for open-ended, sampled k paths for self-consistency, constrained decoding for JSON) and the beam>10 degeneracy; perplexity non-comparability across tokenizers → bits-per-byte; anisotropy (random pairs cosine ~0.6–0.9, calibrate thresholds, only within-model rankings); NER first-subword alignment + entity-level seqeval; verbatim-span+offset verification for LLM extraction; left padding for batched decoder generation; `do_sample=False` swallowing temperature; encoder-distillation architecture for high-volume classification with explicit calibration; don't lowercase/stem/strip for subword models, dedup as the highest-impact corpus op; `max_new_tokens` vs `max_length`; retrieval-vs-map-reduce-vs-long-context decision order and lost-in-the-middle.

## Discipline rules

- Any "model can't do X" claim about strings/numbers/multilingual: show the actual `tokenizer.tokenize()` split of the failing input before theorizing. Two minutes of this replaces hours of prompt voodoo. Recommendation hierarchy for production arithmetic: tool call > digit-formatted prompt > bigger model.
- Any perplexity comparison: same tokenizer, corpus, context length, and windowing protocol — else convert to bits-per-byte or refuse.
- Any extraction system: verbatim-match verification against the source, or outputs are flagged, not trusted.
- Chat prompts only via `tokenizer.apply_chat_template(..., add_generation_prompt=True)`; a missing generation prompt makes the model continue your question — that symptom is diagnostic.

## Sharpenings the strong baseline lacks

- **Multilingual budgeting**: non-English scripts cost 2–10× tokens per unit of meaning on English-centric tokenizers — measure tokens-per-char on a sample before quoting context or cost; never assume ~4 chars/token outside English. Also lower temperature/top-p for non-English generation: thinner tail distributions make sampling riskier.
- **Stop-sequence off-by-one**: stopping on `"\n"` fails when the tokenizer merges `".\n"` into one token — the streamed token text never equals the stop string. Match stops on *detokenized* text or use the API's native stop parameter.
- **`skip_special_tokens=True` silently deletes structure** when the output format uses tokens the tokenizer marks special — check before blaming the model for missing delimiters.
- **Unicode normalization is load-bearing**: NFC vs NFD differences (Vietnamese, Korean, accented European text) fragment tokens invisibly — `unicodedata.normalize("NFC", s)` at ingestion; `ftfy.fix_text` for mojibake (2% mojibake measurably degrades a finetune and poisons embedding clusters).
- **Few-shot delimiter leakage**: a separator that also occurs inside examples (newlines in multi-line answers) makes shot boundaries unparseable — use rare structured delimiters, byte-identical formatting across shots.
- **Asymmetric retrieval prefixes are mandatory, not decorative**: models trained with "query: "/"passage: " prefixes silently lose recall without them — the model card's prefix spec is part of the API. Match cosine-vs-dot to the training metric; normalizing a dot-product-trained model's embeddings changes rankings.
- **Instruction tuning raises web-corpus PPL while improving usefulness** — never rank instruct models by generic-corpus perplexity; also macro-vs-micro document averaging changes PPL, state the protocol.
- **Playground–API quality cliffs**: diff the exact request payloads (template, sampling defaults, model snapshot) before any deeper theory.
- Cross-lingual: a few hundred target-language examples usually beats zero-shot cleverness; run translate-train/translate-test baselines first (often within a point of specialized methods); "amazing zero-shot" results on machine-translated benchmarks are testing translationese.
- BLEU/ROUGE only as legacy secondary numbers for abstractive tasks — n-gram overlap punishes valid paraphrase and rewards degenerate copying.
- Keep raw text immutable; preprocess at load with versioned code — irreversibly preprocessed corpora are unpayable debt when conventions change. `fasttext` lid.176 remains the language-ID workhorse; route language before pipelines, since wrong-language processing fails silently.

## Verification / self-check

1. Diagnosis claims show token splits; decoding recommendations name parameter values matched to task entropy.
2. Perplexity/logprob comparisons normalized to bytes or refused.
3. Similarity designs: contrastively-trained encoder, mandated prefixes applied, threshold calibrated on in-domain random pairs.
4. NER numbers: entity-level seqeval on first-subword alignment, or declared inflated.
5. Extraction: verbatim-location check wired in, non-matching spans dropped/flagged.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found: none major — tokenizer-first diagnosis, anisotropy numbers, left-padding, and verbatim-span grounding all produced cold.
- Retained value: operational sharp edges baseline answers omit (merged-token stop sequences, skip_special_tokens data loss, NFC/NFD fragmentation, retrieval prefix mandates, 2–10× multilingual token inequity as a budgeting number).
