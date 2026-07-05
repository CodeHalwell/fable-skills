---
name: llm-application-engineering
description: Engineering production applications on top of LLM APIs — context budgeting, latency/cost/quality routing, streaming UX, caching, graceful degradation, non-determinism, guardrail layering, token accounting, prompt versioning, and long-document strategies. Load when designing, building, or debugging an LLM-backed product or service (not just a single prompt).
---

# LLM Application Engineering

## Core mental model (anchors — standard discipline, hold it under deadline pressure)

1. An LLM call is an unreliable, expensive, slow RPC: timeouts, `max_tokens`, backoff+jitter, circuit breakers — plus the LLM-specific failure classes: semantically wrong output that parses fine, refusals, format drift, latency scaling with *output* length.
2. Distinct failure paths, coded separately: 429/5xx/timeout → retry with backoff (respect `retry-after`); 400 context-overflow → trim, never retry; refusal (a 200 with refusal-shaped content) → rephrase once or degrade, never identical retry; validation failure → one repair with the error appended, then escalate/fallback. A generic `except: retry` turns a context overflow into an infinite loop.
3. Prefix caching dictates prompt layout: stable first (system, tools, examples), variable last. One dynamic byte early (timestamp, request ID, user name) zeroes the hit rate silently. Response caches key on model + prompt-template version + params + input.
4. Prompts are code: versioned, eval-gated, version-ID in every log line, rolled back like code.
5. Cheapest lever for both latency and cost: make the model say less (schemas, max_tokens, no restating input). Output tokens dominate wall-clock and are priced several× input.
6. Validate referents, not just structure: every model-produced ID, URL, price, or entity your system acts on gets checked against ground truth — schema-valid fabrications are the most damaging production hallucination class because every structural guardrail waves them through.

## The corrections (where default engineering practice falls short)

**Routing is escalate-on-failure, not a one-time model choice.** The reflex is to pick "the best model for the workload" or route by static difficulty tiers. The higher-yield pattern: send traffic to the cheapest plausible tier and *escalate one tier on detected failure* (validation error, refusal, low confidence) — this captures most of the always-use-the-big-model quality at a fraction of the cost, because failures are detectable and rare. Prerequisite: build the router interface (`complete(request, tier)`) on day one even if v1 routes everything to one model — retrofitting routing into raw SDK calls scattered across a codebase is the expensive version.

**A fallback model is not a drop-in.** Failing over from model A to B keeps the service "up" while format, refusal patterns, and quality shift under a prompt and validators tuned for A. Every fallback pair needs its own eval run (often its own prompt variant), and every response must be tagged with the *actual* serving model so downstream metrics can be segmented. Teams test the failover path's plumbing and never its behavior.

**Degradation must be visible or it's unlocalizable.** Fallback answers get shipped unlabeled; six weeks later "quality feels worse" and nobody can say where. Rule: every degraded response is (a) labeled to the user ("partial summary — retry for full"), (b) logged with cause, (c) counted on a per-feature dashboard. Same for mid-stream deaths: detect the stop reason, show a retry affordance, never persist a partial as final — a max_tokens stop on structured output is a guaranteed-broken payload, treat as validation failure not content.

**Write the context budget down before coding.** Everyone "budgets by tokens"; almost nobody assigns each region an owner and an overflow rule. The discipline that works: a table (system, examples, history summary, recent turns, retrieved chunks, input cap, output reserve) where every region has a number and a shrink rule — retrieved chunks shrink first, output reserve never shrinks — and the planned total sits well under the window, because cost and middle-loss scale with usage, not the limit. Test at 100% and 120% of each cap; unbudgeted contexts fail on the longest conversation with the biggest document, silently truncating the thing that mattered.

**Own your conversation state.** Unless deliberately using a provider session feature, every request must be reconstructible from your own store — replay, debugging, provider migration, and evals over real traffic all depend on it. Also: never orphan a tool-result from its tool-call when trimming history (hard API errors), and pin standing user facts outside the trimmable region.

## Compressed operational checklist (one-liners; each is a real incident class)

- max_tokens ≈ 2× expected output and a client timeout on every call site; grep for naked `client.complete`.
- Count tokens with the provider's tokenizer/endpoint, not `len//4` (drifts worst on code, CJK, numbers).
- Separate keys/quotas for batch vs interactive; batch through the provider batch API (~50% cheaper); semaphore + jittered backoff, or 10k asyncio tasks thundering-herd your own rate limit.
- Retries wrap generation only — side effects (send, write, charge) live outside the retry with idempotency keys.
- Guardrails layer: input validation → schema validation (one repair with the error, then fallback) → semantic checks (rules, then judge-to-flag not rewrite) → human escalation with defined triggers. Moderation decisions before streaming starts; retraction reads as creepy.
- Don't stream machine-consumed output; stream user-visible generation >1–2s, buffer to markdown block boundaries.
- Log prompts/completions as a first-class sensitive datastore: redaction before write, retention, access control; check provider retention/training terms.
- Per-feature token attribution and budget alerts from day one; do the cost arithmetic (tokens × current real prices × volume) in design review, not on the invoice.
- Cache only deterministic-intent calls; include prompt version + model in the key or a prompt deploy serves stale behavior.
- A model upgrade or a non-pinned alias is a breaking change: rerun the regression set.

## Worked micro-example — the call wrapper's failure paths (shape)

```python
except RateLimited:      backoff_jitter(retry_after) and retry
except ContextTooLong:   trim and retry (same tier, not counted as attempt)
except Overloaded/5xx:   backoff retry ≤3
if looks_like_refusal(raw):   rephrase-once or degrade+escalate   # refusal ≠ error ≠ bad JSON
if schema_invalid(raw):       repair-once with error → then escalate one TIER → then labeled fallback
log(prompt_version, tier, actual_model, usage)
```
Load-bearing: three *distinct* failure paths, trim-not-retry on overflow, tier escalation as the second retry, version+model logged for attribution.

## Verification / self-check

- Every call site: max_tokens, timeout, three distinct failure paths, defined fallback.
- Context budget table exists with per-region overflow rules; kill-tested at 120% of each cap.
- Cache audit: key completeness; first-divergent-byte check on consecutive prompts confirms the intended stable prefix.
- Kill-test degradation in staging (block the key, force 429s, inject malformed output) — users must see *labeled* degradation.
- Fallback pair has its own eval run; responses tagged with actual serving model.
- Cost model recomputed against last week's real token logs; alert on >20% drift per feature.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 14 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus routes by static difficulty; misses escalate-on-detected-failure as the pattern that beats static routing, and the build-the-router-interface-first discipline.
  - Silent on fallback-model behavioral divergence (failover needs its own eval + serving-model tagging) and on *labeled* degradation as the antidote to unlocalizable quality decline.
  - Knows token budgeting; lacks the written per-region budget with owners/overflow rules (chunks shrink first, output reserve never).
