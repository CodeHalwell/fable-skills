---
name: llm-cost-engineering
description: Token economics and cost optimization for LLM applications — pricing mechanics (input/output asymmetry, caching discounts, batch APIs across Anthropic/OpenAI/Google), the optimization ladder in ROI order, prompt-cache engineering, cheap-model-first routing, output control, RAG-vs-long-context arithmetic, fine-tune break-even math, and cost monitoring with kill-switches. Load when analyzing, forecasting, or reducing LLM API spend, or designing cost-aware architecture.
---

# LLM Cost Engineering

## The standard doctrine, compressed

A strong model already produces this cold; anchors only. Output prices ~5–6× input and reasoning tokens bill as output — "make the model say less" is the cheapest optimization; most production input is repeated prefix, so cache-hit rate is a first-class metric; decisions are per *successful task*, never per MTok (retries, escalations, verbosity, tokenizer differences all live between). Cache mechanics it nails: prefix = exact-byte match, timestamps/UUIDs/unsorted-JSON/per-user tool subsets early in the prompt zero it; Anthropic explicit breakpoints (max 4), writes 1.25× (5-min) / 2× (1-h), break-evens 2 and 3 reads; verify via `usage` fields, never assume. Parallel fan-out of N requests sharing a cold prefix bills N full writes — send one, await first token, release the rest. Agent history is quadratic in turns (cache it, compact it, cap turns); retry loops that resend whole conversations and append full error texts are a cost *and* quality death spiral — retry the failed request only, truncate error feedback. Fine-tune-to-shrink (never to inject knowledge) pays only at sustained volume on a narrow stable task; fine-tuned inference ~1.5–2× base rates and base-model deprecations force re-trains. Kill-switch: client-side hard budget per task counting all four usage fields (input, output, cache write, cache read), limit ≈ p95 × 3–5, halt-and-page on trip; second fuse = feature-level spend-rate anomaly. Token estimation: `len//4` and cross-tokenizer guesses drift 10–30%+ on code/CJK/numbers — use the provider's `count_tokens` against the exact model. Monitoring: tag every request (feature, prompt version, model served, task/user ID + four usage fields); alert on tokens-per-task p95 jumps, effective-$/MTok vs a date-stamped price table, cache-hit drops, context-length distribution (long-context tier boundaries: Gemini 3.1 Pro doubles input above 200k).

## The numbers a cold answer gets wrong (as of mid-2026 — re-verify before quoting)

A model answering from training data quotes the **2024–25 price sheet** (Opus $15/$75, GPT-4o $2.50/$10, Gemini 2.5 Pro) and the old cache parameters. Current table:

| Tier | Anthropic | OpenAI | Google |
|---|---|---|---|
| Frontier | Fable 5: $10 / $50 | GPT-5.5: $5 / $30 | Gemini 3.1 Pro: $2 / $12 (≤200k) |
| Workhorse | Opus 4.8: $5 / $25 · Sonnet 5: $3 / $15 | GPT-5.4: $2.50 / $15 | Gemini 3.5 Flash: $1.50 / $9 |
| Cheap | Haiku 4.5: $1 / $5 | GPT-5.4-mini: $0.75 / $4.50 · nano: $0.20 / $1.25 | Gemini 2.5 Flash: $0.30 / $2.50 |

Spread ~25–50× top-to-bottom — the spread, not any single price, is why routing exists.

Caching corrections to the baseline answer:
- **OpenAI cached input is ~90% off on current flagships** ($0.50 vs $5.00 on GPT-5.5), not the ~50% a cold answer asserts. Automatic, ≥1,024-token prefixes matched in 128-token increments, no write surcharge, ~5–10 min TTL (≤1 h).
- **Anthropic minimum cacheable prefix is model-dependent and larger than remembered:** 4,096 tokens on Opus 4.8, 2,048 on Fable 5 / Sonnet 4.6 — a 3k-token prefix silently never caches (`cache_creation_input_tokens: 0`, no error). Also: max 4 breakpoints, and the cache lookback spans only ~20 content blocks — an agent turn adding 30 tool blocks silently misses the previous turn's entry; place an intermediate breakpoint inside long turns.
- **Gemini:** implicit caching default on 2.5+ (min ~2,048 tokens on 2.5, ~4,096 on 3.x; ~0.1×) plus explicit CachedContent paying **storage rent per token-hour** (~$4.50/MTok/hr on 3.1 Pro, ~$1 on Flash — a forgotten 500k-token cache is ~$54/day of pure rent).
- **Batch discounts stack with caching** — Anthropic and OpenAI publish batch+cached rates; a shared-prefix batch job runs at ~5–10% of naive interactive cost. The cold answer ("treat batch and cache as alternatives; prefixes go cold inside the window") under-claims: put a breakpoint on the shared prefix inside the batch. All three providers: ~50% off both sides, 24 h window, key results by `custom_id` not position.
- **Intro pricing is a trap for break-even models:** Sonnet 5 launched at $2/$10 through 2026-08-31, then $3/$15 — date-stamp every price and alert when billed effective $/MTok diverges. Tokenizers also move between adjacent versions (Sonnet 5 emits ~30% more tokens than Sonnet 4.6 for identical text) — a "cheaper" model can be quietly more expensive per task.

## Ladder ordering: why routing is rung 3, not rung 1

The baseline reflex ranks "route to a cheaper model" first (biggest multiplier). Rank by *risk-adjusted* ROI instead: (1) **prompt slimming + output control** — hours of pure deletion, no infra, compounds with every later rung (a smaller prompt is cheaper to cache, batch, and route); (2) **caching** — near-free on stable prefixes, only touches input; (3) **routing** — first rung with real quality risk: it requires a regression eval with a numeric pass gate (≥97% agreement class) built *before* any traffic moves, plus escalation-on-observable-failure (validation error, refusal, low confidence → one tier up, one retry) — and end-to-end accounting, since a cheap model failing 20% and escalating can cost more per successful task than the mid tier; (4) **batch** — flat 50% on async traffic (if the workload is mostly async, do it first — simpler than routing); (5) **fine-tune-to-shrink** — last, lifecycle-priced. Stopping rule: compute the ceiling first — $500/mo workloads never repay rung 3+; recompute the design-time model against actual logs monthly, >20% drift means the model or the product shape changed.

## Worked spine

*Support product, $38k/mo, CFO wants half; team proposes "switch everything to a cheaper model."* Attribution before action: 60% agent-assist chat (85% of it input cost — a 22k prefix resent per turn), 25% nightly summarization, 15% misc. Reject the model swap as the first move (flagship-surface quality risk, weeks of eval, doesn't touch spend structure — park as rung 3). Move the nightly job to batch (−12.5% total, a day, zero risk). Diff two rendered prompts: byte 214 diverges — `Current time: …` in the system prompt; move it to the end, add a breakpoint after the KB excerpt, watch `cache_read_input_tokens` go nonzero (−~40% total for two lines). Cap outputs (900-token answers customers skim → `max_tokens: 700` + length guidance, −4%). ≈55%: target met without touching model choice; the router stays parked until the *new* baseline justifies it. The arithmetic pattern generalizes — e.g. 150k-token corpus, 500 q/day on a $5/MTok model: stuffed $0.75/query, stuffed-*cached* ~$0.075, RAG@4k $0.02 — cached long-context is within ~4× of RAG (not the pre-caching 37×), so stuff-and-cache when the corpus fits and queries cluster within TTL; RAG when corpus exceeds context, is per-user (no shared prefix), or traffic is sparse.

## Verification / self-check

- All prices verified this session, date-stamped "as of"; arithmetic uses the four-way token split (recompute one example: total prompt = input + cache_creation + cache_read).
- Comparison is per successful task including retries, escalations, and reasoning tokens.
- Cache claims validated against nonzero `usage` reads; routing claims validated by an eval pass rate; every cost lever names the eval gating its quality side.
- `stop_reason: max_tokens` on structured output treated as a validation failure, not content.
- Stopping rule: next rung's projected savings < its loaded engineering cost → stop and say so.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 8 baseline (cut/compressed — cache-miss causes, TTL break-evens, fan-out cold-write herd + prime-first fix, quadratic agent history, kill-switch with all four usage fields + second fuse, tokenizer pitfalls, fine-tune lifecycle, silent-explosion inventory), 3 partial (sharpened), 2 delta (expanded).
- Biggest baseline gaps: the entire price/cache-parameter table is a generation stale in cold answers (2024-25 prices; OpenAI cached-read quoted at 50% vs actual ~90% off; Anthropic minimum-cacheable sizes wrong); batch×cache stacking denied ("treat as alternatives") when providers publish stacked rates; ladder ordering puts routing first without pricing its eval prerequisite and escalation costs.
- Kept as likely-delta despite unprobed: 20-content-block cache lookback, Sonnet 5 intro-pricing expiry and tokenizer shift, Gemini storage-rent dollar figures.
