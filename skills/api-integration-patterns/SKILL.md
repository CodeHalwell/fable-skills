---
name: api-integration-patterns
description: Load when building or reviewing code that consumes or exposes APIs — HTTP clients, retries/timeouts/circuit breakers, rate-limit handling, webhooks (sending or receiving), pagination, API versioning, SDK-vs-raw-HTTP choices, or debugging flaky third-party integrations.
---

# API Integration Patterns

## Core mental model

- **The network is an unreliable dependency you don't control; code accordingly by default.** Every outbound call needs a timeout, a retry policy decision (including "no retry"), and an answer to "what does the user see when this is down?" — decided at write time, not incident time. The un-configured default of most HTTP libraries (infinite or very long timeout, no retry) is the wrong default for production.
- **Retries are a loaded weapon.** They convert partial outages into total ones (retry storms), duplicate non-idempotent operations (double-charged customers), and hide bugs behind eventual success. Retry only what is safe (idempotent or idempotency-keyed) and only what is retryable (timeouts, 429, 5xx — never 4xx except 408/429).
- **At-least-once delivery is the universal contract.** Networks give you "definitely maybe once or more." Both sides of every integration therefore need idempotency: senders attach stable keys; receivers dedupe on them. Exactly-once is achieved at the application layer or not at all.
- **Integrations fail slowly and silently more often than loudly.** The taxonomy that actually bites: expired credentials, silent schema drift in responses, clock skew breaking signatures, pagination edge cases, and a provider changing behavior without changing the version number. Availability monitoring misses all five.
- **Your integration's contract is what the provider *does*, not what the docs say.** Test against reality (sandbox, recorded traffic) and keep the recordings fresh.

## The resilient-client checklist

Every production HTTP client gets, in this order of importance:

1. **Timeouts, split by phase**: connect timeout short (1–3 s — TCP connect either works fast or won't), read timeout sized to the endpoint's real p99 (often 10–30 s; streaming/LLM endpoints need per-chunk read timeouts instead of total). One number for both is a smell. `requests.get(url, timeout=(3, 15))`; in httpx, `httpx.Timeout(connect=3, read=15, write=5, pool=5)`. No timeout = a thread/connection leak waiting for an outage.
2. **Retries with exponential backoff + full jitter**, on idempotent operations only: delay = `random(0, min(cap, base * 2**attempt))`. Full jitter (not "equal jitter", not fixed backoff) is what prevents synchronized retry waves after a blip. 3–4 attempts max; retry on connect errors, 408, 429, 500/502/503/504; never on other 4xx (they will fail identically forever).
3. **Idempotency keys for unsafe operations you must retry**: generate a UUID per logical operation (not per attempt), send it (`Idempotency-Key` header where supported — Stripe-style; many payment/LLM APIs support this), and only then is retrying a POST safe.
4. **Retry budget**: cap retries globally (e.g., retries ≤ 10–20% of request volume, token-bucket implemented). Per-request retry logic looks innocent; at fleet scale, 3 retries × N callers is a 4× load multiplier aimed at a service that is already dying. When the budget is exhausted, fail fast — this is the difference between "degraded" and "cascading outage."
5. **Circuit breaker** for dependencies on the request path: after K consecutive (or %-threshold) failures, open — fail immediately without calling — then half-open with probe requests. Purpose: shed load from the dying dependency *and* protect your own latency budget. Skip breakers for offline/batch callers; a queue with backoff does the same job more simply.
6. **Concurrency cap per dependency** (connection-pool limit or semaphore): unbounded parallelism turns *your* traffic spike into *their* outage and yours.

## Rate-limit engineering

**Client side** — reasoning chain: *What does the provider publish?* → set a local **token bucket** at ~80–90% of the documented limit (leave headroom for other consumers/clock skew) so you throttle yourself smoothly instead of slamming into 429s. *When a 429 arrives anyway*: honor `Retry-After` exactly (seconds or HTTP-date — parse both), don't count it against error budgets as a failure (it's flow control), and reduce send rate rather than merely delaying the one request — a 429 means the *bucket* is empty, so pausing one request while 50 others fly changes nothing. Cap concurrency independently of request rate: 10 requests/s at 60 s each is 600 in flight.

**Server side** (as of 2026): the IETF standard (`draft-ietf-httpapi-ratelimit-headers`, at draft -11 as of May 2026) is still an Internet-Draft, not an RFC; it defines structured `RateLimit` and `RateLimit-Policy` fields. Deployed reality remains the older de facto trio `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` (or the un-prefixed variants from earlier drafts). Practical guidance: emit the de facto headers your clients' SDKs already parse, always send `Retry-After` on 429, document the window semantics precisely (sliding vs. fixed; per-key vs. per-IP vs. per-org), and rate-limit by authenticated principal, not IP, wherever possible. As a client, treat `X-RateLimit-Remaining` as advisory (it races against your own concurrent requests) and the 429 as authoritative.

## Webhook engineering — both directions

**Receiving** (the four non-negotiables):
1. **Verify signatures** before parsing: HMAC over the *raw request body* (bytes as received — recompute over re-serialized JSON and verification breaks on key ordering/whitespace), constant-time comparison. Check the timestamp header and reject if older than ~5 minutes — that's the replay-attack defense; a valid signature on a captured request is replayable forever without it. The **Standard Webhooks** spec (adopted as of 2026 by OpenAI, Anthropic, Twilio, Supabase, and others; most providers still proprietary) standardizes exactly this: `webhook-id`, `webhook-timestamp`, `webhook-signature` (HMAC-SHA256) — use its reference libraries when the provider complies.
2. **Respond fast, process async**: return 2xx after persisting the raw event to a queue/table — do the work later. Senders time out in seconds and treat timeouts as failures → redelivery → duplicate processing of slow handlers. Never do business logic inline.
3. **Dedupe on the delivery/event ID** (at-least-once is guaranteed to bite): `INSERT ... ON CONFLICT DO NOTHING` on the event ID before processing.
4. **Don't trust event payloads for state**: treat the webhook as a hint ("something changed about object X") and re-fetch the authoritative object when the stakes are high — payloads can arrive out of order (delivery order is not creation order under retries).

**Sending**: sign (per Standard Webhooks if greenfield), deliver at-least-once with a documented retry schedule (exponential over hours-to-a-day, e.g. 8–12 attempts), include a unique message ID for consumer dedup, and solve **the dead-webhook problem**: endpoints that 4xx/timeout for days. Auto-disable after a sustained failure window (e.g., 3–7 days), notify the owner out-of-band, and provide a redelivery/replay API so recovered consumers can catch up — otherwise every consumer outage becomes a support ticket asking you to replay by hand. Never follow redirects on delivery, and pin/validate the destination against SSRF if URLs are user-supplied.

## Polling vs. webhooks vs. streaming

Decision chain: *Who owns the state, and how fresh must the consumer be?*
- Freshness in minutes+ and volume modest → **poll** with `If-None-Match`/cursor; it's self-healing (no missed-event problem), trivially debuggable, and often all you need. Don't build webhook infrastructure to check something hourly.
- Freshness in seconds, provider offers webhooks → **webhooks + reconciliation polling**: the webhook is the fast path; a slow poll (hourly/daily listing) catches dropped deliveries. Webhooks without reconciliation silently diverge — every mature integration ends up adding the poll after the first missed-event incident, so design it in.
- Sustained high-volume or bidirectional → **streaming** (SSE for server→client simplicity, WebSocket for bidirectional, Kafka-class for firehoses). Streaming buys freshness at the cost of connection lifecycle management (resume tokens, heartbeats, redelivery on reconnect) — budget for that code; a stream without resume handling is a webhook with extra steps.

## Pagination client patterns

- **Prefer cursor/keyset pagination**; treat offset pagination on mutable data as broken by design — inserts/deletes between pages cause silent skips and duplicates (**the moving-dataset problem**). If offset is all the provider offers, either sort by an immutable monotonic key and filter (`?created_after=<last seen>` — keyset smuggled through query params) or accept and dedupe.
- Cursors are opaque: never parse or construct them, never assume they survive beyond the provider's documented TTL. Handle "cursor expired" (often 400/410 mid-crawl) by restarting the listing from a checkpoint you control (last processed immutable key), not from scratch.
- Loop hygiene: terminate on absent/empty `next` cursor AND a hard page cap (a provider bug that returns the same cursor forever should not hang your job); dedupe on primary key across pages; persist the checkpoint every page so a crash resumes instead of re-crawling.

## Exposing APIs: what your consumers' failure modes teach you

Everything above, mirrored — design your API so the resilient-client checklist is *possible*:
- Support idempotency keys on unsafe operations and document retention (24h is a common window); return the original response on replay, not an error.
- Return 429 with `Retry-After` (never a bare 429, never a 500 for rate limiting); document the limit's principal and window semantics.
- Make pagination cursor-based from day one — you cannot retrofit cursor stability onto a public offset API without a version bump.
- Additive changes only within a version: new fields and new enum values are *your* prerogative, so say loudly in docs that clients must ignore unknown fields and handle unknown enum values; breaking changes get a new version with `Deprecation`/`Sunset` headers on the old one, months of overlap, and usage-based outreach to laggards before shutoff.
- Emit a machine-readable request ID on every response and log it — "can you give me the request ID" is the first question in every support escalation, in both directions.

## Versioning, SDK vs. raw HTTP, and testing

- **Pin API versions explicitly** (header or URL) — never ride "latest". Subscribe to the provider's changelog mechanically (deprecation headers like `Deprecation`/`Sunset` exist — log and alert on them; most teams discover deprecations from the outage). Calendar the sunset dates.
- **SDK vs. raw HTTP** reasoning: take the official SDK when it's actively maintained and handles auth/retry/pagination for you (that's real, tested code you don't write). Go raw (or thin-wrap) when the SDK lags the API, when its retry/timeout behavior is opaque or unconfigurable (an SDK with hidden infinite retries breaks your retry budget), or when it drags heavyweight/conflicting dependencies. Either way, wrap the dependency behind your own interface — the failure-handling policy must be yours, and swapping later must not touch call sites.
- **Testing strategy**: unit tests against *recorded real responses* (VCR-style cassettes), not hand-written mocks — hand mocks encode your assumptions, which is exactly what's wrong when the integration breaks (**mock drift**). Refresh cassettes on a schedule or on provider version bumps. Contract tests: a small suite that hits the provider's sandbox in CI (nightly, not per-commit — sandboxes are flaky) asserting the response *shapes* you depend on. Sandbox caveats: sandboxes lag production behavior and rarely simulate rate limits or webhook retries — test those paths with fault injection in your own stack.

## How an expert thinks through it: "the integration is flaky"

Payments-provider calls fail ~2% of the time, "randomly." Internal monologue: *"Random" usually means "correlated with something we're not logging." First: what does a failure look like — connect timeout, read timeout, 5xx, 429? Get the taxonomy before theorizing.* Logs show mostly read timeouts, clustered at :00–:05 past each hour. *Clustered → not random → something scheduled. Ours or theirs? Our cron dashboard: a reconciliation job fires hourly and fans out ~500 concurrent calls through the same connection pool (limit 50). The interactive traffic isn't failing because the provider is slow; it's queueing behind our own batch burst for pool connections, then hitting read timeout.* Rejected en route: blaming provider capacity (would not align to *our* clock); adding retries (would add load to the exact contended window — retries fix transient faults, not systematic contention); raising the timeout (hides the queueing, worsens tail latency). Fix: separate connection pool + concurrency cap + token-bucket pacing for the batch job, spread its schedule with jitter, and add per-phase timeout + pool-wait metrics so the next contention is visible directly. *Also check the 429 handling while in here — the batch job was treating 429 as a hard failure and re-queueing items for the next hour, compounding the burst.* Stopping rule: a week of taxonomy-labeled error metrics showing failures at baseline (<0.1%) with no hourly pattern; not "it looks better today."

## Failure modes & pitfalls

- **No timeout / one merged timeout.** Symptom: worker pools drain during a dependency brown-out. Correction: explicit connect+read everywhere; lint for `timeout=None` (note `requests` defaults to no timeout; `httpx` defaults to 5 s — know your library's default instead of assuming).
- **Retrying POSTs without idempotency keys.** The classic double-charge. Correction: keys per logical operation, or don't retry — reconcile instead.
- **Retry on every status code.** Retrying 400/401/403/404 burns budget on guaranteed failures and can lock accounts (401 retries against a lockout counter). Correction: explicit allowlist (408, 429, 5xx, connect errors).
- **Backoff without jitter.** All clients that failed together retry together — waves at t+1s, t+2s, t+4s. Correction: full jitter.
- **Ignoring `Retry-After` and hammering on 429**, escalating throttling into IP bans. Correction: parse both header forms; respect it; back off globally, not per-request.
- **Signature verification over parsed-then-reserialized JSON.** Works in dev (same serializer), breaks in prod. Correction: HMAC over raw bytes captured before any body parsing middleware; beware frameworks that consume the body stream.
- **No timestamp check on webhooks** → replay attacks with validly signed captured payloads. Correction: reject deliveries older than ~5 min; require the timestamp inside the signed content (Standard Webhooks does).
- **Webhook handler doing synchronous work** → sender timeout → redelivery → duplicates → the team "fixes" it by removing dedup ("we never get duplicates") → duplicates. Correction: persist-and-ack in <1 s; process from the queue; dedupe on event ID unconditionally.
- **Assuming webhook ordering.** `object.updated` can arrive before `object.created` under retries. Correction: idempotent handlers keyed on object state (fetch current), or sequence numbers when provided.
- **Offset pagination over live data** in a nightly export: rows silently skipped when deletes shift pages. Correction: keyset on immutable key; verify export counts against a totals endpoint.
- **Auth expiry as a surprise**: OAuth refresh tokens expiring from disuse, API keys rotated by a teammate, certs expiring. Correction: monitor auth failures as their own alert class (a 401 spike is an ops event, not an error blip); refresh proactively before expiry; alarm on token age.
- **Clock skew breaking signed requests** (AWS SigV4-style and webhook timestamps tolerate minutes at most). Symptom: everything 403s on one misconfigured host. Correction: NTP everywhere; include skew in the debugging taxonomy.
- **Silent schema drift**: provider adds an enum value or nulls a field your code assumed present; nothing 4xxs. Correction: validate responses at the boundary (Pydantic/zod) in *warn* mode with alerting — strict mode turns every benign additive change into your outage; unknown enum values route to an explicit `unknown` branch, never `else: assume old behavior`.
- **Token-refresh stampede**: an access token expires and 200 in-flight workers simultaneously hit the refresh endpoint — some providers invalidate the refresh token on concurrent use, locking the whole integration out. Correction: single-flight the refresh (mutex/distributed lock), refresh proactively at ~80% of token lifetime, and serialize refresh-token rotation.
- **200-with-errors blindness**: GraphQL and batch endpoints return HTTP 200 with per-item/partial failures in the body; `raise_for_status()` sees success. Correction: the boundary wrapper inspects the body's error envelope and converts partial failures into typed results — never let "HTTP succeeded" stand in for "operation succeeded."

## Worked micro-example: resilient client core (Python, httpx + tenacity)

```python
import httpx, uuid
from tenacity import retry, stop_after_attempt, wait_random_exponential, retry_if_exception

TIMEOUT = httpx.Timeout(connect=3.0, read=15.0, write=5.0, pool=5.0)
LIMITS = httpx.Limits(max_connections=50, max_keepalive_connections=20)  # concurrency cap
client = httpx.Client(timeout=TIMEOUT, limits=LIMITS)

RETRYABLE = {408, 429, 500, 502, 503, 504}

def _should_retry(exc: BaseException) -> bool:
    if isinstance(exc, (httpx.ConnectError, httpx.ReadTimeout)):
        return True
    return isinstance(exc, httpx.HTTPStatusError) and exc.response.status_code in RETRYABLE

@retry(stop=stop_after_attempt(4),
       wait=wait_random_exponential(multiplier=0.5, max=30),  # full jitter, capped
       retry=retry_if_exception(_should_retry), reraise=True)
def create_payment(payload: dict, idem_key: str) -> dict:
    r = client.post("https://api.example.com/v2/payments", json=payload,
                    headers={"Idempotency-Key": idem_key})       # same key across attempts
    if r.status_code == 429 and (ra := r.headers.get("Retry-After")):
        # honor server pacing before tenacity's own backoff
        import time; time.sleep(min(float(ra), 30))
    r.raise_for_status()
    return r.json()

# Call site: key per LOGICAL operation, minted once, stored with the order
result = create_payment(payload, idem_key=str(order.payment_attempt_uuid))
```

Missing here by design (add per system): retry budget (global token bucket around `create_payment`), circuit breaker for request-path use, and response-shape validation.

## Worked micro-example: webhook receiver done right (FastAPI)

```python
import hmac, hashlib, base64, time
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
TOLERANCE_S = 300  # 5 min replay window

def verify_standard_webhook(secret: bytes, msg_id: str, ts: str, raw_body: bytes, sig_header: str):
    if abs(time.time() - int(ts)) > TOLERANCE_S:
        raise HTTPException(400, "stale timestamp")            # replay defense
    signed = f"{msg_id}.{ts}.".encode() + raw_body             # sign RAW bytes, id+ts included
    expected = base64.b64encode(hmac.new(secret, signed, hashlib.sha256).digest()).decode()
    # header may carry multiple space-separated versioned sigs: "v1,<b64> v1,<b64>"
    candidates = [p.split(",", 1)[1] for p in sig_header.split() if p.startswith("v1,")]
    if not any(hmac.compare_digest(expected, c) for c in candidates):  # constant-time
        raise HTTPException(401, "bad signature")

@app.post("/webhooks/provider")
async def receive(request: Request):
    raw = await request.body()                                 # BEFORE any JSON parsing
    verify_standard_webhook(SECRET, request.headers["webhook-id"],
                            request.headers["webhook-timestamp"], raw,
                            request.headers["webhook-signature"])
    # Dedupe + persist + ack. No business logic here.
    inserted = await db.execute(
        "INSERT INTO webhook_events (id, raw, received_at) VALUES ($1,$2,now()) "
        "ON CONFLICT (id) DO NOTHING", request.headers["webhook-id"], raw)
    if inserted:
        await queue.enqueue("process_webhook", request.headers["webhook-id"])
    return {"ok": True}                                        # 2xx in <1s either way
```

The four non-negotiables are all present and separable in review: raw-body HMAC with constant-time compare, timestamp window, ID-keyed dedup, ack-then-process. For a provider on the Standard Webhooks spec, replace `verify_standard_webhook` with the official `standardwebhooks` library — but the shape stays identical for proprietary schemes (Stripe, GitHub) with different header names and signed-string construction: always find what exactly is signed (raw body alone vs. id.timestamp.body) in the provider's docs, never guess.

## Verification / self-check

Before shipping an integration:
1. Kill test: point at a blackhole address (e.g., a firewalled port) — does the call fail within your timeout, or hang?
2. Duplicate test: run the same logical operation twice with the same idempotency key / deliver the same webhook twice — exactly one side effect?
3. 429 test: fault-inject a 429 with `Retry-After: 7` — does the client wait ~7 s and reduce rate, or hammer?
4. Replay test: re-send a captured webhook 10 minutes later — rejected?
5. Pagination test: mutate the collection mid-crawl in sandbox — any skipped/duplicated items?
6. Chaos hour: block the provider at the firewall for 10 minutes in staging — do circuit breakers open, does the user-facing degradation match the plan, does recovery happen unassisted?

Stopping rule: when the six tests above pass and error metrics are labeled by taxonomy (timeout/429/5xx/auth/schema), stop hardening. Resilience beyond that (hedged requests, regional failover for the dependency) is warranted only when the SLO math — dependency availability vs. your promised availability — says the gap exists.
