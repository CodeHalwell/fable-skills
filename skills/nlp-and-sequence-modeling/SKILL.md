---
name: nlp-and-sequence-modeling
description: Load for NLP and language-model engineering — diagnosing tokenizer-caused model failures, BPE vs unigram choices, picking decoding strategies (greedy/beam/nucleus/temperature) per task, interpreting perplexity correctly, embedding-similarity pitfalls, sequence labeling and span extraction, multilingual/cross-lingual work, and long-sequence handling.
---

# NLP and Sequence Modeling

## Core mental model

- **The tokenizer is the model's sensory organ; many "model" mysteries are tokenizer facts.** LMs never see characters or digits — they see subword IDs. Arithmetic errors, spelling/reversal failures, "strawberry has two r's," whitespace sensitivity, and multilingual inequity are mostly explained at the tokenization layer. When an LM behaves bizarrely on string manipulation, numbers, or a low-resource language, inspect the token boundaries *first*.
- **A language model defines a distribution; decoding is a separate policy choice on top of it.** The same model is deterministic-precise or creative-diverse depending on decoding. Many "the model is bad at X" complaints are "the decoding strategy is wrong for X."
- **Perplexity is exp(mean NLL per token) and is only meaningful relative to a fixed tokenizer and corpus.** Cross-tokenizer perplexity comparisons are numerically meaningless without renormalization.
- **Embedding spaces are trained for specific objectives; similarity is only meaningful within that objective's geometry.** Cosine similarity between arbitrary vectors from a model not trained with a similarity objective is weakly meaningful at best.
- **Prefer span/label formulations over free generation when the output is constrained.** Extraction, tagging, and classification tasks done via unconstrained generation inherit hallucination; done as classification/span-selection they can't invent content.

## Tokenization: the root of many mysteries

Failure signatures and their tokenizer explanations:
- **Arithmetic**: numbers split inconsistently ("2023" one token, "2024" maybe two: "202","4"); digit-place alignment is invisible. Models with digit-split tokenization (every digit its own token) do measurably better at arithmetic. Before concluding "can't do math," check how the specific numbers tokenize.
- **Spelling/reversal/counting letters**: the model sees `["straw","berry"]`, not letters. Character-level tasks require the model to have *memorized* each token's spelling. Workaround that actually works: force character separation in the prompt ("s-t-r-a-w-b-e-r-r-y").
- **Whitespace sensitivity**: `"hello"` and `" hello"` are different tokens with different learned statistics; a prompt ending in a trailing space forces the next token to be a rare no-leading-space variant → measurably worse continuations. Never end prompts with trailing whitespace; be suspicious of any few-shot template whose examples end mid-token pattern.
- **Multilingual inequity**: a tokenizer trained mostly on English encodes other scripts at 2–10× more tokens per unit of meaning → less effective context, higher cost, worse quality for the same nominal context window. When budgeting context or cost for non-English text, measure tokens-per-char on a sample; never assume English ratios (~4 chars/token).
- **Code**: indentation and identifier splitting dominate; a tokenizer that merges common whitespace runs (as modern code-aware ones do) massively changes effective context for code.
- Chat-template bugs are tokenization bugs: a missing/extra newline or wrong special token in the template silently degrades an instruct model. Use the model's `tokenizer.apply_chat_template`, never hand-rolled strings.

Debug ritual: `tokenizer.tokenize(s)` / `tokenizer(s).input_ids` on the exact failing string, including leading spaces. Two minutes of this replaces hours of prompt voodoo.

## BPE vs unigram (subword algorithms)

- **BPE**: bottom-up greedy merges of frequent pairs; deterministic segmentation. Frequent strings become single tokens; rare words fragment into arbitrary, non-morphological pieces ("undesirable" → pieces that ignore un+desire+able). Byte-level BPE (GPT-style) guarantees no `<unk>` ever — any byte sequence tokenizes.
- **Unigram LM (SentencePiece default)**: top-down probabilistic — keeps a vocabulary maximizing corpus likelihood; segmentation is the Viterbi-best but *alternatives exist*, enabling subword regularization (sampling segmentations during training) which measurably improves robustness for MT and low-resource setups.
- Behavioral differences that matter: unigram tends to produce more morphologically plausible splits; BPE's greedy merges make token identity hyper-sensitive to frequency (typos and rare inflections fall off a cliff). For training a new tokenizer on morphologically rich or low-resource languages, prefer unigram/SentencePiece with subword regularization; for compatibility with the LLM ecosystem, byte-level BPE is the de facto standard.
- Vocab size tradeoff: bigger vocab → shorter sequences (cheaper attention, more effective context) but bigger embedding table and rarer-token undertraining. 32k–128k is the practical band; going past that is mostly justified for heavily multilingual models.

## Decoding strategies matched to task

| Task type | Strategy | Why |
|---|---|---|
| Closed-form answers: classification-via-LM, extraction, math final answers, code where determinism matters | greedy or temperature→0 | you want the mode; sampling injects errors with no benefit |
| Machine translation, summarization (trained seq2seq) | beam (4–10) with length normalization | short-horizon mode-seeking; beam finds higher-likelihood sequences than greedy; length norm prevents the "beam loves short outputs" bias |
| Open-ended generation: dialogue, stories, brainstorming | nucleus (top-p 0.9–0.95) + temperature 0.7–1.0 | pure mode-seeking here yields degenerate repetition ("the the the" attractors); truncated sampling cuts the unreliable tail while preserving diversity |
| Diverse candidates for reranking (best-of-n, self-consistency) | temperature ~0.7–1.0 sampling, n samples | you want coverage of the answer distribution; self-consistency (majority vote over sampled reasoning) reliably beats greedy on reasoning tasks |
| Constrained output (JSON, grammar) | constrained decoding (logit masking via grammar/JSON-schema enforcement) | sampling hopes for validity; masking guarantees it — use `outlines`/native structured-output features, not regex-retry loops |

Mechanics worth stating precisely: temperature rescales logits *before* softmax (T<1 sharpens, T>1 flattens); top-p truncates to the smallest set with cumulative prob ≥ p then renormalizes; top-k is a fixed-size truncation (worse than top-p when the distribution's entropy varies by position). Repetition penalties are a blunt instrument that also punishes legitimately repeated entities — prefer better sampling before reaching for them.
Pitfall: very long beams (>10) on open-ended tasks *degrade* output (generic, repetitive, short) — beam search's assumption (mode = quality) fails when the target distribution is high-entropy.

## Perplexity: meaning and non-comparability

- PPL = exp(Σ NLL / N_tokens). It is per-token; the denominator's tokenizer defines the number.
- **Never compare PPL across tokenizers/vocabs**: a tokenizer producing fewer, bigger tokens gets *higher* per-token PPL for identical modeling quality (each token carries more information). To compare models with different tokenizers, normalize to bits-per-byte or bits-per-character: total NLL in bits ÷ byte count of the raw text — invariant to tokenization.
- Also non-comparable across: different eval corpora (obviously), different context lengths (longer context → lower PPL), and sliding-window vs full-context evaluation protocols.
- PPL measures fit to the corpus distribution, not usefulness: instruction tuning typically *raises* PPL on generic web text while improving task performance. Never rank instruct models by web-corpus PPL.
- Doc-level subtlety: averaging per-token NLL over concatenated documents vs per-document then macro-averaging gives different numbers; state the protocol.

## Embedding-space reasoning

- **Use embeddings trained for similarity** (contrastively trained sentence encoders) for retrieval/similarity. Mean-pooling an off-the-shelf LM's hidden states gives mediocre similarity geometry; raw hidden states are trained for next-token prediction, not for cosine-comparability.
- **Anisotropy**: LM hidden states occupy a narrow cone — random sentence pairs get cosine ~0.6–0.9. Consequences: absolute cosine values are meaningless ("0.8 similar!" may be baseline); only *rankings within the same model* carry signal, and even those degrade if you skipped a similarity-trained model. Never compare cosine values across models, and never set an absolute cosine threshold without calibrating on in-domain random pairs.
- Asymmetric retrieval (short query → long doc) needs a model trained for it (and often prompt prefixes like "query: "/"passage: " that the model card mandates — omitting them silently costs recall).
- Cosine vs dot product: match the metric the model trained with (check the card); normalizing embeddings for a dot-product-trained model changes rankings.
- Chunking dominates retrieval quality more than the embedding model choice: chunk to semantically coherent units (~100–500 tokens) with overlap; embedding a 4-page doc as one vector averages away everything specific.

## Sequence labeling and span extraction

- Tagging (NER, PII, chunking): encoder + per-token classifier with BIO/BILOU scheme. Two classic traps: (1) **subword alignment** — label only each word's first subword (or pool subwords); mislabeling continuation subwords corrupts training and inflates token-level metrics; (2) **evaluate at entity level** (exact span + type, `seqeval`), never token accuracy — 99% token accuracy is compatible with 60% entity F1 under heavy O-class imbalance.
- Invalid transitions (I-PER after B-ORG) — either add a CRF layer or post-hoc repair; report whether metrics are computed pre- or post-repair.
- Span extraction (QA-style start/end pointers) beats generation for extraction whenever the answer must be verbatim from the source: it cannot hallucinate, gives calibrated confidences, and is cheap. If using an LLM for extraction, force verbatim quoting + string-match verification against the source; unverified generative extraction *will* paraphrase and invent.
- LLM-for-NER pattern that works: generate structured JSON of (entity, type, verbatim_text), then locate each verbatim_text in the source; drop non-matching spans. This converts hallucination into a detectable failure.

## Task → approach selection (encoder finetune vs LLM)

| Task | Default approach | Switch when |
|---|---|---|
| Text classification, high volume, fixed labels | finetuned small encoder (DeBERTa/ModernBERT class): cheap, fast, calibrated probabilities | < ~500 labeled examples or labels shift often → LLM few-shot/zero-shot; use the LLM to *bootstrap labels*, then distill to the encoder for serving |
| NER / PII / tagging, production | finetuned encoder + BIO (+ verbatim rules for high-precision types like emails/regex-able IDs) | open-ended entity types or no training data → LLM with verbatim-quote verification (pattern below) |
| Semantic search / dedup / clustering | contrastively trained embedding model + ANN index | reranking quality matters → add a cross-encoder reranker over top-k; cross-encoders beat bi-encoders on accuracy but can't scale to the full corpus |
| Summarization / rewriting / open generation | instruction-tuned LLM | never a small seq2seq from scratch unless domain is extremely narrow and latency-critical |
| Pairwise similarity with a threshold decision | cross-encoder classifier trained on pairs | embeddings + cosine only for the candidate-generation stage |

Cost/latency rule: if a task runs > ~100k times/day with fixed structure, the LLM is the *teacher*, not the server — generate silver labels, train the small model, keep the LLM for the tail. Calibration rule: encoder classifiers give usable probabilities after temperature scaling; LLM token-choice "probabilities" from sampled text are not calibrated class probabilities — if you need thresholds, get logprobs of the label tokens or train a proper classifier.

## Cross-lingual transfer realities

- Multilingual encoders transfer zero-shot (train on English, apply to language X) surprisingly well between typologically close, well-represented languages, and poorly for low-resource/distant ones — quality tracks the language's pretraining share and tokenizer efficiency (see inequity above).
- A few hundred target-language examples usually beats elaborate zero-shot tricks; translate-train (MT the training set) and translate-test are strong baselines to run before anything clever — often within a point of specialized methods.
- Label-projection tasks (NER across languages) fail on word-alignment errors more than modeling; entity boundaries shift across translations.
- Beware evaluation contamination in "amazing zero-shot" claims: multilingual benchmarks are frequently machine-translated from English, so the model is being tested on translationese that matches translate-train pipelines.

## Long-sequence handling

- Decision order for input longer than the effective window: (1) does the task actually need the whole document, or is it retrieval in disguise? → chunk + retrieve (RAG) handles most QA/extraction; (2) map-reduce (per-chunk process, then combine) for summarization-like tasks — mind that hierarchical summarization compounds omissions; (3) true long-context model only when cross-chunk reasoning is genuinely global (long-range dependencies, whole-codebase reasoning).
- Effective context < nominal context: retrieval quality degrades with distance and with middle placement ("lost in the middle" behavior — critical info at the extremes of the prompt outperforms buried-in-middle). Place instructions and the most important context at the beginning or end; verify with position-swept probes on your own task.
- Chunking for labeling long docs with encoders: sliding windows with overlap ≥ max entity length; dedupe entities in the overlap region by offset.
- Cost sanity: attention prefill grows ~quadratically; a 100k-token prompt per query in a high-QPS service is an architecture smell — cache shared prefixes (system prompt / document) if the stack supports prefix caching, or restructure to retrieval.

## Failure modes & pitfalls

- **Hand-building chat prompts instead of `tokenizer.apply_chat_template(messages, add_generation_prompt=True)`** — a missing `add_generation_prompt` leaves the model completing the user turn instead of answering; symptom: outputs that start by continuing your question.
- **Comparing logprobs across models with different tokenizers** (e.g., for ensembling or reranking) — per-token logprobs are not on a common scale; sum to sequence level and normalize by bytes, or don't compare.
- **`max_length` vs `max_new_tokens` confusion** in `generate()`: `max_length` includes the prompt, so long prompts silently truncate generation to near zero. Always use `max_new_tokens`.
- **Truncating from the wrong end**: default `truncation=True` cuts the tail; for tasks where the answer/instructions sit at the end (chat, QA with question-last), that deletes the question. Set truncation side deliberately; better, fail loudly on overflow.
- **Padding-side bugs for decoder-only models**: batched generation requires *left* padding (`tokenizer.padding_side = "left"`); right padding puts pad tokens between prompt and generation, garbling outputs for all but the longest sequence in the batch.
- **Reporting "similarity 0.83" as if it were a probability** — see anisotropy; without an in-domain random-pair baseline the number is uninterpretable.
- **Few-shot format leakage**: examples separated by a delimiter that also appears inside examples (newlines in multi-line answers) — the model can't tell where examples end; use rare, structured delimiters and keep formatting byte-identical across shots.
- **Casing/accent mismatch with the tokenizer's training**: an uncased model fed cased text, or NFC vs NFD Unicode normalization differences (especially Vietnamese, Korean, accented European text) fragmenting tokens — normalize (`unicodedata.normalize("NFC", s)`) at ingestion.
- **Assuming `skip_special_tokens=True` is safe** when the output format uses tokens the tokenizer considers special — structured outputs lose delimiters silently.
- **Evaluating generation with BLEU/ROUGE where semantics matter**: n-gram overlap punishes valid paraphrases and rewards degenerate copying; for anything abstractive, use model-based/judge eval plus targeted string checks, and report ROUGE only as a secondary legacy number.
- **Stop-sequence off-by-one**: stopping on `"\n"` for a model whose tokenizer merges `".\n"` into one token means the stop never matches the streamed text; match stops on detokenized text, not token IDs, or use the API's native stop parameter.

## Worked micro-example: diagnosing a "the model can't do X" report

Report: "the model fails at adding 4-digit numbers but its bigger sibling succeeds." Expert diagnosis path: (1) tokenize the failing inputs — `tokenizer.tokenize("3487+2915=")` reveals whether digits are split as `["348","7","+","29","15","="]` (misaligned place values) or per-digit; (2) reformat the prompt to force per-digit tokens: `"3 4 8 7 + 2 9 1 5 ="` and re-test — if accuracy jumps, the deficit is tokenization, not arithmetic capability, and the fix is formatting or tool use, not a bigger model; (3) if unchanged, sweep decoding — greedy vs sampled at T=0.7 (sampling injects digit errors; arithmetic should always be evaluated greedy); (4) only after (1)–(3) conclude anything about the model. The same path applies to spelling, rhyming, and string-reversal complaints. Recommendation hierarchy for production arithmetic: tool call (calculator/code) > digit-formatted prompting > model scaling.

## Verification / self-check

1. Any string/number/multilingual failure diagnosis: show the actual token split (`tokenizer.tokenize`) supporting it.
2. Any decoding recommendation: confirm it matches the task's entropy profile (closed-form → mode-seeking; open-ended → truncated sampling) and mention the parameter values.
3. Any perplexity claim: same tokenizer? same corpus? same context protocol? If not, convert to bits-per-byte or refuse the comparison.
4. Any cosine-similarity design: is the encoder contrastively trained for this use, are mandated prefixes applied, and is the threshold calibrated against in-domain random-pair baseline?
5. Any extraction system: is there a verbatim-match verification step? If output can't be located in the source, it must be flagged, not trusted.
6. Any NER metric: entity-level F1 via `seqeval` on word-first-subword alignment — confirm both, or the number is inflated.
