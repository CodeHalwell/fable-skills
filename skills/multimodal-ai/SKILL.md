---
name: multimodal-ai
description: Engineering with vision, audio, document, and video models — image token/cost arithmetic, preprocessing (crop/tile/zoom), PDF and document-AI pipeline choices, structured extraction with grounding, voice-agent latency architecture (STT/LLM/TTS vs speech-to-speech), video frame sampling, and evaluating multimodal outputs. Load when building anything that feeds images/PDFs/audio/video to a model or generates images/speech.
---

# Multimodal AI Engineering

## Core mental model

1. **Know the capability map before designing the pipeline.** As of 2026, frontier VLMs are genuinely good at: chart/figure reading, UI understanding (element recognition, screenshot-driven computer use), OCR-ish reading of clean Latin-script text, table structure, handwriting that's legible to humans, and holistic "what is this image" reasoning. They still reliably fail at: **precise counting** beyond ~10 similar objects, **fine spatial precision** (exact pixel coordinates, sub-element alignment, compositional spatial relations), small/low-contrast text near the resolution floor, non-Latin and degraded-document OCR, and negation over visual scenes ("which chair has NO cushion"). Design so the model does what it's good at and *code* does the rest: detect-then-count with CV, crop-then-read, overlay-grid-then-locate.
2. **Images are tokens; do the arithmetic before the architecture.** Anthropic: image tokens ≈ `(width × height) / 750` (a 1000×1000 image ≈ 1,334 tokens); images beyond the model's max edge get downscaled first. OpenAI's vision pricing is tile-based (a base cost plus per-512px-tile cost in high-detail mode; exact numbers vary per model — check current docs, don't recite from memory). Two consequences: (a) a 50-page PDF rendered at high DPI is easily 100k+ tokens — budget first; (b) *downscaling destroys small text before it destroys layout*, so token savings and OCR accuracy trade directly.
3. **Resolution is a budget you aim, not a dial you max.** The highest-leverage multimodal technique is preprocessing: crop the region that matters and send it at native resolution, rather than the full page at whatever survives downscaling. A two-pass pattern — cheap low-res pass to locate, high-res crop pass to read — routinely beats one max-res pass on both cost and accuracy.
4. **Latency architecture for voice is a pipeline-sum problem.** Human turn-taking tolerance is roughly 500–800ms voice-to-voice. Budget it: streaming STT (~150–300ms after endpoint), LLM time-to-first-token (~200–400ms), streaming TTS time-to-first-byte (~100–200ms). Every component must *stream*; one batch stage blows the budget. The alternative is native speech-to-speech models — lower latency and prosody-aware, at the cost of component swapability and (often) weaker tool-use control.
5. **Multimodal outputs need grounding to be trustworthy.** An extraction without a pointer back to the source (page, bounding box, timestamp) can't be audited, and VLMs hallucinate plausible values from context (reading a total where none is printed). Structured extraction should carry citations wherever the stack supports it, and confidence-critical fields should be verified by an independent read.

## Decision frameworks

### The capability map — route work to model vs code (state of practice, mid-2026)

| Task | VLM alone | Expert routing |
|---|---|---|
| Chart/figure QA, trend reading | Good | VLM; ask for the underlying numbers table first, then reason over it |
| UI/screenshot understanding | Good (drives computer-use agents) | VLM; for clicking, use models/APIs with explicit grounding support |
| Clean printed text (Latin) | Good | Text layer if it exists (free + exact); VLM otherwise |
| Complex tables | Decent, degrades with density | Layout parser first; VLM verify/repair on failures |
| Handwriting (legible) | Good | VLM — now beats classical OCR here |
| Non-Latin / degraded scans | Weak-to-fair | Purpose-built OCR models; measure per-script |
| Counting >~10 similar objects | Unreliable | Detector counts; VLM classifies |
| Precise coordinates / measurements | Unreliable | Detection/segmentation models, or grounding-specific APIs; verify by re-rendering |
| Negation, absence ("which has NO…") | Unreliable | Reframe as positive enumeration + code filters |
| Small text near resolution floor | Fails silently | Crop to native resolution — no prompt fixes sub-~20px glyphs |

The rows change slowly; the columns' quality improves every model generation — re-verify the "unreliable" rows against the current frontier before designing around a limitation, but never *assume* one has been fixed.

### Structured extraction from visual inputs

- Always schema-guided (tool call / JSON schema), never free text you re-parse. Every field optional/nullable with an explicit "null if not visible" instruction — required fields are a fabrication pump.
- Grounding: where the stack supports citations/bounding boxes (provider citation features, grounding-tuned VLMs), demand them and *verify by rendering* — draw the box, check the value is inside it. Where it doesn't, require a verbatim source snippet per field and string-match it against the OCR/text layer; a snippet that doesn't appear in the source is a hallucination flag you can automate.
- Order the schema to mirror reading order of the document; extraction quality measurably improves when the model fills fields in the order it encounters them.
- Split mega-schemas: one pass for header fields, one per table/region. A 60-field single pass degrades tail-field accuracy; two 30-field passes on crops usually cost less than the retries.

### Document pipeline: parse natively, render-to-image, or hybrid?

Questions in order:
1. **Is there a reliable text layer?** Digital-born PDFs → text extraction (fast, cheap, exact) with a parsing pipeline for structure. Scans/photos → visual path required.
2. **Does layout carry meaning?** Multi-column, tables, forms, stamps, checkboxes → naive text extraction shreds reading order; you need either a layout-aware parser or a VLM look.
3. **Volume vs fidelity?** As of 2026 the landscape has three tiers: (a) **document parsing pipelines** that emit structured Markdown/JSON — Docling, MinerU, LlamaParse, Mistral OCR — best for high-volume ingestion (RAG corpora) where per-page cost matters; (b) **OCR-purpose-built compact VLMs** — olmOCR, Surya — self-hostable, cheap per page; (c) **frontier VLM reads the rendered page directly** — highest fidelity on messy layouts, handwriting, and "answer a question about this page," most expensive per page.
4. **Default hybrid for mixed corpora:** parse everything cheaply; route pages that fail heuristics (garbled text ratio, empty extraction, table density, low OCR confidence) to the VLM path. Most corpora are 90% easy pages; pay the VLM only for the hard 10%.
5. Provider-native PDF input (e.g., Anthropic's PDF support) does render+text-layer for you — convenient and a solid default, but you pay tokens for both the image and extracted text of each page, so it's for *analysis of documents in a conversation*, not bulk ingestion.

### Audio: chained pipeline vs native speech-to-speech

- **Chained (streaming STT → LLM → streaming TTS):** choose when you need best-in-class parts you can swap, text-domain guardrails/logging, complex tool use, or strict control over what gets said. This is still the default for production voice agents in 2026 (vendors like AssemblyAI, Deepgram, ElevenLabs sell the parts and increasingly the assembled pipeline behind one API).
- **Native speech-to-speech (OpenAI Realtime, Google Gemini Live, Hume EVI):** choose when latency and prosody dominate (emotional nuance, natural interruptions) and the task is conversational rather than transactional. Costs: harder to guardrail (no text checkpoint mid-pipeline unless you add one), model lock-in, tool-calling maturity varies.
- The interruption problem (barge-in) is a *product requirement*, not a nicety: you need always-on VAD/endpointing, TTS playback you can kill instantly, and state reconciliation ("how much of my sentence did the user actually hear before cutting me off?") — decide what the LLM's history should say it said. Semantic endpointing (is the user done or just pausing?) is the current quality frontier; a fixed silence-timeout either interrupts thinkers or feels laggy.

STT selection judgment (the spec-sheet WER number is nearly useless):
- WER concentrates where it hurts: proper nouns, domain jargon, alphanumeric IDs, accented speech, crosstalk. Benchmark on *your* audio — 50 real clips with ground truth beats any leaderboard; slice results by accent and SNR.
- Streaming vs batch are different products with different models/accuracy: don't validate on batch and ship streaming.
- Check the features that decide integration pain before accuracy deltas of 1%: word-level timestamps (needed for grounded citations into audio), diarization quality (meeting use cases live or die on it), custom vocabulary/keyword boosting (the fix for domain-term WER), endpointing controls, and price per streamed hour.
- Realistic mid-2026 expectations: streaming latencies around 150–300ms and mid-single-digit WER on clean English from the major vendors (Deepgram, AssemblyAI, ElevenLabs, OpenAI, Google); differences on *your* domain audio dwarf differences on their marketing benchmarks.

### Video

Almost all practical "video understanding" is frame sampling + VLM. Economics dominate: at 1 fps, a 10-minute video is 600 frames — hundreds of thousands of tokens. Expert defaults: sample 0.5–1 fps for activity understanding; use shot/scene detection to pick representative frames instead of uniform sampling; transcribe audio separately (it usually carries most of the semantic load — and is ~100× cheaper per minute); reserve dense sampling for the specific segment a cheap pass located. Ask "what question am I answering?" — "did the assembly step get skipped" needs the 20 relevant seconds densely, not the hour uniformly.

### Generation-side (brief, as of mid-2026)

The image-gen market is multi-model by design: GPT Image 2 for instruction-heavy generation/editing and reasoning over layout; FLUX.2 family for photorealism with open weights (LoRA/fine-tune ecosystem); Imagen 4 for natural photographic looks; Ideogram/Seedream when *legible in-image text* is the requirement. Route per request-type rather than picking one winner; text rendering is still the sharpest differentiator, so test your actual text cases before committing.

### Evaluating multimodal outputs

- Build the golden set before the pipeline: 100–300 hand-labeled examples covering every input condition you'll meet (digital/scan/photo; accents/noise; chart types). Label at the *field* level, not document level — "83% of documents perfect" hides "the total field is wrong 15% of the time."
- Normalize before scoring: currency/number/date canonicalization for extraction, text normalization (casing, punctuation, number formatting) before WER for speech. Un-normalized exact-match understates real quality and hides real errors equally.
- Use LLM-as-judge only for what rules can't score (caption quality, summary faithfulness), with a rubric per criterion and periodic human calibration on a sample; use deterministic checks (schema validity, arithmetic consistency, citation-snippet match, box-contains-value) for everything else — they're free and unimpeachable.
- Track the *fallback and human-queue rates* as first-class metrics: a pipeline whose accuracy "improved" by routing 30% of traffic to humans didn't improve, it moved cost.
- Re-run the golden set on every model/prompt/DPI change. Multimodal pipelines are exquisitely sensitive to preprocessing changes that look harmless (a default DPI bump, a new JPEG quality setting) — treat preprocessing config as versioned code under eval, not as environment.

## How an expert thinks through it

*Scenario: extract line items (description, qty, unit price, total) from 40k supplier invoices/month — mixed digital PDFs, scans, occasional photos.*

First instinct to resist: "send every page to the frontier VLM." Arithmetic: an invoice page at readable resolution is ~1.5–2.5k image tokens plus prompt/output; at 40k docs/month that's real money and, worse, no grounding. (Rejected as the *universal* path, kept as the fallback path.)

Second instinct to resist: classic OCR + regex. Invoice layouts are unbounded; template-per-supplier is a maintenance treadmill. (Rejected.)

Chosen shape: router + two paths. Path A (digital PDFs, ~70%): text+layout extraction via a parsing pipeline; feed structured text to a cheap LLM with the extraction schema — no vision tokens at all. Path B (scans/photos): render at ~150–200 DPI, VLM with schema-guided extraction. Router: does the PDF text layer yield >X chars with sane word confidence? Then A, else B.

Hallucination control, because finance: every extracted field must carry a source snippet; then *code*, not the model, checks `qty × unit_price ≈ line_total` and `Σ line_totals ≈ invoice_total`. Arithmetic consistency is a free, high-precision hallucination detector on numeric documents — a fabricated field almost never balances. Rows failing checks go to a second pass: crop the table region (from layout parse or a cheap locate pass), re-read the crop at native resolution. If still inconsistent → human queue.

Why crops? A photographed invoice downscaled to fit the model's max edge loses exactly the small digits I care about; a crop of the line-item table keeps them at native resolution for a tenth of the full-page tokens.

Eval before scale-up: 200 hand-labeled invoices, field-level precision/recall, sliced by path (digital/scan/photo) and by supplier. Stopping rule: ship when critical numeric fields hit target precision with the human-queue rate affordable; do not chase the last 2% with prompt tweaks — add suppliers to the eval and fix the *router and preprocessing* first, because that's where the errors will actually live.

## Failure modes and pitfalls

- **Sending full pages when the answer lives in a region.** The #1 accuracy fix is a crop. If the model misreads small text, check what resolution that text was at *after provider downscaling* — if the glyphs are under ~15–20px tall post-resize, no prompt will save you. Locate → crop → re-ask.
- **Trusting VLM counts and coordinates.** "How many bolts in this tray?" and "give me the bounding box" both produce confident wrong answers. Counting: use detection (or grid-overlay prompting at minimum) and let code count. Coordinates: only trust models/features explicitly trained for grounding, and verify by drawing the box back onto the image programmatically before believing it.
- **Ignoring image token cost until the bill.** Budget per document *before* building: pages × tokens-per-page at your DPI. Anthropic ≈ `w×h/750`; provider max-edge limits silently downscale oversized inputs (check current limits — they've risen across model generations; don't recite a stale ceiling).
- **Resizing/compressing with the wrong tool for the content.** JPEG-compressing a screenshot smears 1px UI text; aggressive downscale before upload (to save bandwidth) pre-destroys what the provider resize would have kept. Send PNG for screenshots/documents; do your own resize only if you've verified target legibility.
- **Naive PDF text extraction on layout-heavy pages.** Raw `pdftotext`-style extraction interleaves columns and flattens tables; the LLM then confidently answers from shredded text. If tables matter, you need layout-aware parsing or the visual path — check a sample's extraction *by eye* before trusting the corpus.
- **Batch components in a "realtime" voice stack.** One non-streaming stage (waiting for full LLM output before TTS starts) turns 700ms into 3s+. Stream everything; start TTS on the first sentence; measure p95 voice-to-voice, not component means. And handle barge-in from day one — retrofitting interruption into a half-duplex design is a rewrite.
- **STT word errors compounding in the LLM.** Domain terms, names, and alphanumerics (order IDs) are where WER concentrates. Use vendor keyword-boost/custom-vocabulary features, confirm critical strings back to the user ("that's order A-1-9-2?"), and never let a voice agent take an irreversible action off a single unconfirmed transcription.
- **Schema-free extraction.** Free-text answers about images invite hallucinated fields. Use structured output (tool call / JSON schema) with `null` explicitly allowed and prompted for absent fields — models fabricate values to satisfy required fields; a nullable schema plus "use null if not visible" measurably cuts fabrication. Then validate cross-field consistency in code.
- **Evaluating multimodal with text-only metrics.** Exact-match punishes benign OCR variance ("$1,000.00" vs "1000.00"); normalize before scoring. For generation, arena-style human preference and *task-specific* checks (did the text render? is the count right?) beat FID-style scores for product decisions. Always slice evals by input condition (scan vs digital, accent, audio SNR, chart type) — aggregate scores hide the failing slice that dominates your support tickets.

## Worked micro-examples

**Token/cost budget for a document job (Anthropic-style accounting):**
```
Invoice page rendered at 150 DPI  →  1275×1650 px
tokens ≈ 1275 × 1650 / 750 ≈ 2,805 per page
40,000 docs × 1.4 pages avg × 2,805 ≈ 157M image tokens/month
→ at $/Mtok input pricing this is the dominant cost line; routing 70%
  of pages to the text path cuts it ~70% before any model tuning.
```

**Two-pass locate-then-read (Python sketch):**
```python
# Pass 1: cheap, low-res — locate the region
low = render_page(pdf, page, dpi=72)
region = ask_model(low, "Return which quadrant contains the line-item table",
                   schema=Quadrant)          # coarse location only — never exact px

# Pass 2: native-res crop — read it
hi = render_page(pdf, page, dpi=200)
crop = hi.crop(quadrant_bbox(region, hi.size))
items = ask_model(crop, EXTRACT_PROMPT, schema=LineItems)  # nullable fields!

# Code-level verification — the model never checks its own math
for it in items.rows:
    assert it.qty is None or abs(it.qty * it.unit_price - it.total) < 0.01, it
```

**Voice loop budget (chained, streaming):**
```
user stops speaking ──► endpoint detect   ~150–300ms   (streaming STT, VAD)
                    ──► LLM first token   ~200–400ms   (short system prompt, no cold RAG)
                    ──► TTS first audio   ~100–200ms   (websocket TTS, sentence-chunked)
                                          ≈ 450–900ms voice-to-voice; target p95 < 800ms
Rule: anything that adds a blocking step (retrieval, tool call) needs a spoken
filler strategy ("let me check that…") — silence past ~1s reads as broken.
```

## Verification and self-check

- For any extraction claim, spot-check by *rendering the evidence*: draw the cited region/box on the image, or play the cited audio span, and confirm the value is actually there. If your pipeline can't produce that evidence view, it can't be audited — fix that first.
- Re-run the numeric/cross-field consistency checks on a sample by hand; a pipeline whose outputs balance arithmetically is your strongest cheap signal.
- Before quoting any provider limit, price, or token formula in a design, verify against current provider docs this session — image pricing, max resolutions, and audio model names have churned repeatedly through 2024–2026.
- Stopping rule: the pipeline is done when the failing slices are identified, routed to fallback or humans, and the residual error rate is priced into the product — not when the aggregate metric stops improving. Past that, effort goes to preprocessing and routing, not to bigger models.
