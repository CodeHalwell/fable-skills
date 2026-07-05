---
name: api-integration-patterns
description: Load when building or reviewing code that consumes or exposes APIs — HTTP clients, retries/timeouts/circuit breakers, rate-limit handling, webhooks (sending or receiving), pagination, API versioning, SDK-vs-raw-HTTP choices, or debugging flaky third-party integrations.
---

# API Integration Patterns

Frontier models already produce the resilient-client canon cold — split timeouts, full-jitter retries with allowlisted statuses, retry budgets, idempotency keys, webhook HMAC-over-raw-bytes + replay windows, cursor pagination, token-refresh single-flight. This sheet keeps the checklists for review completeness plus the 2026 standards status and calibrations a cold model lacks.

## The resilient-client checklist (one line each; expand only when challenged)

1. Timeouts split by phase: connect 1–3 s, read sized to real p99 (10–30 s; per-chunk for streaming/LLM endpoints), plus pool-acquisition. `requests` defaults to *no* timeout; `httpx` defaults to 5 s *per phase*, not total.
2. Full-jitter exponential backoff (`random(0, min(cap, base·2^attempt))`), 3–4 attempts, retry only connect errors/408/429/5xx — never other 4xx.
3. Idempotency key per *logical operation* (not per attempt) before retrying any unsafe POST.
4. Fleet-level retry budget (retries ≤ 10–20% of volume, token bucket) — per-request retries at fleet scale are a load multiplier aimed at a dying service.
5. Circuit breaker for request-path dependencies; a queue with backoff does the job more simply for batch callers.
6. Concurrency cap per dependency, independent of request rate: 10 req/s at 60 s each = 600 in flight.

## Rate limits

Client: local token bucket at ~80–90% of the documented limit; on 429 honor `Retry-After` (parse both seconds and HTTP-date), reduce *send rate* globally rather than delaying the one request, and don't count 429 as an error-budget failure. `X-RateLimit-Remaining` is advisory (races your own concurrency); the 429 is authoritative. Watch the reset-header trap: some APIs send seconds-until-reset, others epoch — blind `sleep(reset)` on an epoch sleeps 50 years.

Server (status as of 2026 — cold models overstate maturity): the IETF `draft-ietf-httpapi-ratelimit-headers` is at **draft -11 (May 2026), still an Internet-Draft, not an RFC**, defining structured `RateLimit`/`RateLimit-Policy` fields. **Deployed reality remains the de facto `X-RateLimit-*` trio — emit those** (they're what clients' SDKs parse), always `Retry-After` on 429, document window semantics precisely, and rate-limit by authenticated principal, not IP. Emitting only the draft fields today serves almost no client.

## Webhooks

Receiving — four non-negotiables (all four, separable in review): HMAC over the **raw body bytes** with constant-time compare; timestamp check (~5 min) as the replay defense; dedupe on event ID (`ON CONFLICT DO NOTHING`) before processing; persist-and-ack in <1 s, process async. Plus: treat payloads as hints and re-fetch authoritative state when stakes are high — delivery order is not creation order under retries.

**Standard Webhooks adoption (2026 fact):** the spec (`webhook-id`/`webhook-timestamp`/`webhook-signature`, HMAC-SHA256 over `{id}.{timestamp}.{body}`) is now adopted by **OpenAI, Anthropic, Twilio, and Supabase** among others — not just the Svix ecosystem a cold model remembers. Use its reference libraries when the provider complies; the verify shape is identical for proprietary schemes (Stripe/GitHub) — always find *what exactly is signed* in the docs, never guess. Signature-header parsing detail: the header may carry multiple space-separated versioned sigs (`v1,<b64> v1,<b64>`); verify against any.

Sending: sign (Standard Webhooks if greenfield), documented exponential retry over ~24–72 h (8–12 attempts), unique message ID, and solve the dead-endpoint problem — auto-disable after a sustained failure window (3–7 days), notify out-of-band, and ship a replay API so recovered consumers catch up. Never follow redirects on delivery; validate destinations against SSRF.

Polling vs webhooks: minutes-fresh + modest volume → poll (self-healing, debuggable). Seconds-fresh → webhooks **+ reconciliation polling designed in from day one** — every mature integration adds the poll after its first missed-event incident anyway. Streaming only for sustained volume/bidirectional, and budget for resume tokens/heartbeats — a stream without resume handling is a webhook with extra steps.

## Pagination

Cursor/keyset always; offset over mutable data is broken by design (skips + duplicates). Offset-only provider → smuggle keyset through `?created_after=<last seen>` on an immutable sort, or dedupe. Cursors are opaque with TTLs — handle "cursor expired" mid-crawl by restarting from *your own* checkpoint (last processed immutable key). Loop hygiene: hard page cap (a same-cursor-forever provider bug must not hang the job), per-page checkpoint persistence, PK dedupe across pages.

## Exposing APIs (mirror image, compressed)

Idempotency keys with documented retention (return the original response on replay); 429 + `Retry-After`, never a bare 429 or a 500; cursor pagination from day one (you cannot retrofit cursor stability without a version bump); additive-only within a version — tell clients loudly to ignore unknown fields and route unknown enum values to an explicit `unknown` branch; `Deprecation`/`Sunset` headers + months of overlap + usage-based outreach; request ID on every response.

## Versioning, SDK vs raw, testing

Pin versions; alert on `Deprecation`/`Sunset` headers (most teams discover deprecations from the outage). SDK when maintained and its retry/timeout behavior is transparent; raw/thin-wrap when the SDK lags, hides infinite retries (breaks your retry budget), or drags conflicting deps — either way wrap behind your own interface. Test against recorded real responses (VCR cassettes, refreshed on schedule), not hand mocks (mock drift encodes exactly the assumptions that are wrong); nightly contract tests against the sandbox asserting response *shapes*; fault-inject rate limits and webhook retries in your own stack because sandboxes don't simulate them.

## Debugging calibration: "flaky" means "correlated with something unlogged"

Get the failure taxonomy (connect timeout / read timeout / 5xx / 429) before theorizing. Time-clustered failures → something scheduled; check *your own* crons before blaming provider capacity (an hourly fan-out of 500 calls through a 50-connection pool makes interactive traffic queue behind your own batch, then read-timeout). Rejected reflexes: adding retries (adds load to the contended window — retries fix transient faults, not systematic contention); raising the timeout (hides the queueing). Fix: separate pool + concurrency cap + pacing + jittered schedule; and check the 429 path while in there — re-queueing 429'd items to the next hour compounds the burst. Stopping rule: a week of taxonomy-labeled metrics at baseline, not "looks better today."

## Silent-failure checklist (what availability monitoring misses)

Expired/rotated credentials (alert on 401 spikes as an ops class; refresh proactively at ~80% of lifetime; **single-flight the refresh** — concurrent refresh-token rotation can trigger reuse-detection and lock out the whole integration); silent response-schema drift (validate at the boundary in *warn* mode — strict mode turns benign additive changes into your outage); clock skew breaking signatures (NTP; minutes of tolerance at most); pagination truncation; 200-with-errors bodies (GraphQL/batch endpoints — the wrapper inspects the error envelope; `raise_for_status()` is not "operation succeeded").

## Worked micro-example: webhook receiver skeleton (FastAPI)

```python
import hmac, hashlib, base64, time
from fastapi import FastAPI, Request, HTTPException

TOLERANCE_S = 300

def verify_standard_webhook(secret: bytes, msg_id: str, ts: str, raw_body: bytes, sig_header: str):
    if abs(time.time() - int(ts)) > TOLERANCE_S:
        raise HTTPException(400, "stale timestamp")            # replay defense
    signed = f"{msg_id}.{ts}.".encode() + raw_body             # RAW bytes, id+ts in signed string
    expected = base64.b64encode(hmac.new(secret, signed, hashlib.sha256).digest()).decode()
    candidates = [p.split(",", 1)[1] for p in sig_header.split() if p.startswith("v1,")]
    if not any(hmac.compare_digest(expected, c) for c in candidates):
        raise HTTPException(401, "bad signature")

@app.post("/webhooks/provider")
async def receive(request: Request):
    raw = await request.body()                                 # BEFORE any JSON parsing middleware
    verify_standard_webhook(SECRET, request.headers["webhook-id"],
                            request.headers["webhook-timestamp"], raw,
                            request.headers["webhook-signature"])
    inserted = await db.fetchrow(                              # execute() returns "INSERT 0 0" (truthy) on skip
        "INSERT INTO webhook_events (id, raw, received_at) VALUES ($1,$2,now()) "
        "ON CONFLICT (id) DO NOTHING RETURNING id", request.headers["webhook-id"], raw)
    if inserted is not None:                                   # row returned only when actually inserted
        await queue.enqueue("process_webhook", request.headers["webhook-id"])
    return {"ok": True}                                        # 2xx in <1s either way
```

## Verification / self-check (the six tests)

1. Blackhole a dependency — fails within your timeout, or hangs?
2. Same logical operation twice with the same key / same webhook twice — exactly one side effect?
3. Injected `429 Retry-After: 7` — waits ~7 s and reduces rate, or hammers?
4. Captured webhook replayed 10 min later — rejected?
5. Collection mutated mid-crawl — skips or duplicates?
6. Provider firewalled 10 min in staging — breakers open, degradation matches plan, unassisted recovery?

Stop hardening when these pass and error metrics are taxonomy-labeled. Hedged requests/regional failover only when the SLO math says the gap exists.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Opus cold nails: split timeouts + library defaults, full-jitter + retry budget, raw-byte HMAC + replay window, reconciliation polling, offset-pagination corrections, 429 discipline incl. epoch-vs-seconds trap, dead-endpoint auto-disable, token-refresh stampede + rotation lockout.
- Sharpened: IETF rate-limit draft status pinned (draft -11, May 2026, still not RFC — emit the de facto `X-RateLimit-*` trio, cold models over-recommend the draft fields); Standard Webhooks adopter list updated to OpenAI/Anthropic/Twilio/Supabase (cold knowledge stops at the Svix ecosystem).
