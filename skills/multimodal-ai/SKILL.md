---
name: multimodal-ai
description: Engineering with vision, audio, document, and video models — image token/cost arithmetic, preprocessing (crop/tile/zoom), PDF and document-AI pipeline choices, structured extraction with grounding, voice-agent latency architecture (STT/LLM/TTS vs speech-to-speech), video frame sampling, and evaluating multimodal outputs. Load when building anything that feeds images/PDFs/audio/video to a model or generates images/speech.
---

# Multimodal AI Engineering

Assumed baseline (verified expert-grade cold): the VLM capability map (good at chart/UI/clean-OCR/handwriting; fails at counting >~8, precise coordinates, small text, dense tables, negation, non-Latin OCR) and routing geometry-to-code/semantics-to-VLM; image tokens ≈ w×h/750 (1000² ≈ 1334) and downscale-before-tokenize; resolution-as-budget with the two-pass locate-then-read pattern; nullable schemas as fabrication control; arithmetic-consistency as the cheap numeric hallucination detector; routed digital-text-vs-VLM invoice pipeline; ~500–800ms voice tolerance with streaming component budgets and barge-in/endpointing as the hard problem; WER concentrating on proper nouns/IDs with keyword-boost/timestamps mattering more than 1% deltas; frame-sampling + audio-transcript economics for video; golden-set + normalization + per-slice/per-field eval; document-parsing tiers (deterministic text layer → OCR → layout parsers → frontier VLM).

## Discipline rules

- Do the token/cost arithmetic before the architecture: pages × tokens-per-page at your DPI. A 50-page PDF at high DPI is easily 100k+ tokens; naive 1 fps video is ~4.7M tokens/hour/question.
- Route work to the model's strengths and let *code* do the rest: detect-then-count, crop-then-read, overlay-grid-then-locate, and verify any extraction by rendering the evidence (draw the box / play the span) — a pipeline that can't produce that evidence view can't be audited.
- Numeric documents: code checks `qty×unit_price ≈ line_total` and `Σ ≈ total` — the model never checks its own math; a fabricated field almost never balances.
- Re-run the golden set on every model/prompt/**DPI/JPEG-quality** change — multimodal pipelines are exquisitely sensitive to preprocessing config; treat it as versioned code under eval, not environment.

## The hard-floor fact and the numbers to pin

- **Glyph size is a hard floor, not a soft one**: below ~**10 px cap-height as the model sees it** (after provider downscale), reading collapses and *no prompt recovers it* — the information is gone at the tokenizer input. Target ~16–20 px. Compute glyph height after downscale (original px × resized/original), and remember a 300-DPI A4 scan (~8.7 MP) gets shrunk ~3× against the ~1.15 MP / ~1568-px-edge cap before tokenization. This converts "the model misreads small text" from a prompting problem into a tiling/DPI problem.
- **Verify every provider constant against current docs this session** — the /750 divisor, the ~1.15 MP cap, max edges, per-tile pricing, and audio model names have churned repeatedly through 2024–2026. Never recite a stale ceiling into a design or a customer quote.

## Sharpenings the strong baseline lacks

- **Schema ordering and splitting**: order fields to mirror the document's reading order (extraction quality measurably improves when the model fills fields in encounter order); split mega-schemas — one pass for header fields, one per table/region. A 60-field single pass degrades tail-field accuracy; two 30-field passes on crops usually cost less than the retries.
- **Grounding is verify-by-rendering, not trust-the-citation**: where the stack supports boxes/citations, draw the box and confirm the value is inside it; where it doesn't, require a verbatim source snippet per field and string-match it against the OCR/text layer — a snippet absent from the source is an automatable hallucination flag.
- **Provider-native PDF input pays tokens for both the rendered image and the extracted text of each page** — a solid default for *analysis of a document in a conversation*, wrong for bulk ingestion. For mixed corpora, parse everything cheaply and route only pages failing heuristics (garbled-text ratio, empty extraction, table density, low OCR confidence) to the VLM path — most corpora are 90% easy pages.
- **Chained voice stack is still the production default in 2026** (swappable best-in-class parts, a text checkpoint for guardrails/logging, mature tool use); native speech-to-speech wins on latency/prosody but is harder to guardrail (no mid-pipeline text checkpoint unless you add one) and has less mature tool-calling. Barge-in requires always-on VAD, instantly-killable TTS, and state reconciliation ("how much did the user actually hear before cutting me off?") — deciding what the LLM's history says it said. Semantic endpointing (done vs pausing) is the current quality frontier; a fixed silence timeout either interrupts thinkers or feels laggy.
- **Send PNG for screenshots/documents**: JPEG-compressing a screenshot smears 1px UI text; aggressive pre-upload downscale (to "save bandwidth") pre-destroys what the provider resize would have kept. Do your own resize only after verifying target legibility.
- **STT streaming vs batch are different products/models** — don't validate on batch and ship streaming; slice WER by accent and SNR on 50 real clips with ground truth (beats any leaderboard).
- **Track fallback/human-queue rate as a first-class metric**: a pipeline whose accuracy "improved" by routing 30% to humans didn't improve, it moved cost. Normalize (currency/number/date/text) before scoring or exact-match understates real quality; label at the *field* level ("83% of documents perfect" hides "the total is wrong 15% of the time").
- **Image-gen market is multi-model by request-type** (mid-2026): GPT-image / Gemini-Imagen-class for legible in-image text and prompt adherence, FLUX family for open-weight/self-host/fine-tune, Midjourney for aesthetics, Firefly for rights-cleared enterprise, Ideogram for typography. Text rendering is the sharpest differentiator — test your actual text cases before committing. Route per request rather than picking one winner.

## Verification / self-check

1. Extraction claims spot-checked by rendering the cited region/box or playing the cited span.
2. Numeric/cross-field consistency checks rerun by hand on a sample — arithmetic balance is the strongest cheap signal.
3. Provider limits/prices/token formulas verified against current docs this session before quoting.
4. Golden set sliced by input condition (digital/scan/photo, accent, SNR, chart type); fallback/human-queue rate reported alongside accuracy.

Stopping rule: done when failing slices are identified and routed to fallback/humans and residual error is priced into the product — not when the aggregate metric stops improving. Past that, effort goes to preprocessing and routing, not bigger models.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta (baseline independently flagged the token constants as worth re-verifying, matching the skill's own caution).
- Biggest baseline gaps found: none factually wrong — Opus produced the capability map, /750 formula, glyph-floor, arithmetic-check detector, and voice budgets cold; it was vaguer on schema ordering/splitting and grounding-by-rendering.
- Retained value: the hard-floor glyph number as an actionable threshold, verify-provider-constants discipline (constants drift), schema-ordering/splitting mechanics, and the current-generation image-gen routing map.
