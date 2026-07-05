---
name: multimodal-ai
description: Engineering with vision, audio, document, and video models — image token/cost arithmetic, preprocessing (crop/tile/zoom), PDF and document-AI pipeline choices, structured extraction with grounding, voice-agent latency architecture (STT/LLM/TTS vs speech-to-speech), video frame sampling, and evaluating multimodal outputs. Load when building anything that feeds images/PDFs/audio/video to a model or generates images/speech.
---

# Multimodal AI Engineering

Assumed baseline (verified expert-grade cold): Anthropic image tokens ≈ w×h/750 with ~1.15 MP/1568px downscale caps (1000×1000 ≈ 1,334 tokens); the VLM capability map (good: charts, UI, clean OCR, legible handwriting; unreliable: counting >~10, exact coordinates, sub-~15–20px glyphs, negation/absence, non-Latin/degraded OCR) and route-to-code fixes (detector counts, grounding models for geometry, crop for small text — no prompt fixes a resolution floor); nullable-fields-with-explicit-null schemas against fabrication; arithmetic cross-field consistency as the free hallucination detector on numeric documents; router + text-path/VLM-path invoice architecture rejecting both VLM-on-everything and template OCR; voice budgets (500–800ms voice-to-voice; streaming STT ~150–300ms, LLM TTFT ~200–400ms, TTS TTFB ~100–200ms), chained-vs-S2S tradeoffs, barge-in/endpointing as the hard product problem; spec-sheet WER uselessness (errors concentrate on names/IDs/jargon; phrase-biasing and timestamps matter first); video = cheap-locate-then-dense-read with absence reframed as positive detection; two-pass locate-then-read cropping as the top accuracy-per-dollar move; golden sets labeled at field level, normalization before scoring, slicing by input condition.

## Landscape facts worth pinning (as of mid-2026; re-verify limits/prices in-session before quoting)

- **Document pipeline tiers**: (a) parsing pipelines emitting structured Markdown/JSON — **Docling, MinerU, LlamaParse, Mistral OCR** — for high-volume ingestion; (b) OCR-purpose-built compact VLMs — **olmOCR, Surya** — self-hostable, cheap per page; (c) frontier VLM on the rendered page — highest fidelity, most expensive. Default hybrid: parse everything cheaply, route pages failing heuristics (garbled-text ratio, empty extraction, table density) to the VLM — most corpora are 90% easy pages.
- **Provider-native PDF input** (e.g. Anthropic's) does render+text-layer for you but **bills tokens for both the page image and its extracted text** — right for in-conversation document analysis, wrong for bulk ingestion.
- **Image generation routes per request-type**: GPT Image 2 (instruction-heavy generation/editing, layout reasoning), FLUX.2 family (photorealism + open weights/LoRA ecosystem), Imagen 4 (natural photographic looks), **Ideogram/Seedream when legible in-image text is the requirement** — text rendering is still the sharpest differentiator; test your actual text cases before committing.
- Chained voice stacks remain the production default (AssemblyAI/Deepgram/ElevenLabs sell parts and assembled pipelines); S2S (OpenAI Realtime, Gemini Live, Hume EVI) wins on latency/prosody but loses text-domain guardrails/logging and tool-use maturity.

## Sharpenings the strong baseline lacks

- **Schema mechanics that measurably move extraction accuracy**: order fields to mirror the document's reading order; split mega-schemas — two ~30-field passes on crops beat one 60-field pass (tail-field accuracy degrades) and usually cost less than the retries.
- **Streaming and batch STT are different products with different models and accuracy** — never validate on batch and ship streaming. Slice STT evals by accent and SNR on ~50 real clips; never let a voice agent take an irreversible action off a single unconfirmed transcription (read back critical alphanumerics).
- **Audio carries most of a video's semantic load at ~100× less cost per minute** — transcribe first, sample frames second; use shot/scene detection over uniform fps.
- **Blocking steps in voice need a spoken filler** ("let me check that…") — silence past ~1s reads as broken; retrofitting barge-in into a half-duplex design is a rewrite, so build kill-able TTS playback and history reconciliation ("how much did the user actually hear?") from day one.
- **Preprocessing is versioned code under eval**: a default DPI bump or new JPEG quality setting silently shifts accuracy — re-run the golden set on every preprocessing change, not just model/prompt changes. Send PNG for screenshots/documents; JPEG smears 1px UI text, and your own aggressive pre-resize destroys what the provider resize would have kept.
- **Track fallback and human-queue rates as first-class metrics** — a pipeline whose accuracy "improved" by routing 30% of traffic to humans moved cost, not quality. Separately track hallucinated-when-absent (false positives on truly-null fields) — the dangerous error class that aggregate accuracy hides.
- Capability-map maintenance: the failure rows improve every model generation — re-verify "unreliable" rows against the current frontier before designing around a limitation, but never *assume* one was fixed.

## Worked anchor: budget before architecture

```
Invoice page @150 DPI → 1275×1650 px → ~2,800 tokens/page (w×h/750)
40k docs × 1.4 pages × 2.8k ≈ 157M image tokens/month → dominant cost line;
routing the ~70% digital-PDF share to the text path cuts it ~70% before any model tuning.
```

Stopping rule: done when failing slices are identified and routed (fallback or human) and residual error is priced in — past that, effort goes to preprocessing and routing, not bigger models.

## Verification / self-check

1. Extraction claims spot-checked by rendering the evidence (draw the cited box / play the cited span); a pipeline that can't produce that view can't be audited.
2. Numeric outputs re-checked by code (`qty × unit_price ≈ line_total`, totals foot) — the model never checks its own math.
3. Glyph height computed *post-downscale* before diagnosing OCR failures; crops verified to stay under the resize caps.
4. Provider limits/prices/token formulas verified against current docs this session — they have churned repeatedly through 2024–2026.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 11 baseline (cut/compressed), 3 partial (sharpened), 0 delta.
- Biggest baseline gaps found: 2026 tool-tier specifics (MinerU/olmOCR/Surya/Mistral OCR; GPT Image 2/FLUX.2/Imagen 4/Seedream naming) and provider-native PDF's image+text double token cost; core engineering judgment (token math, crop-first, arithmetic verification, voice budgets, absence-reframing) was produced cold.
- Retained value: pinned landscape facts, schema-ordering/split mechanics, streaming-vs-batch STT trap, audio-first video economics, fallback-rate accounting.
