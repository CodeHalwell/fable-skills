---
name: llm-application-engineering
description: Engineering production applications on top of LLM APIs — context budgeting, latency/cost/quality routing, streaming UX, caching, graceful degradation, non-determinism, guardrail layering, token accounting, prompt versioning, and long-document strategies. Load when designing, building, or debugging an LLM-backed product or service (not just a single prompt).
---

# LLM Application Engineering

## Core mental model

1. **An LLM call is an unreliable, expensive, slow RPC — architect accordingly.** Everything you know about flaky network services applies: timeouts, retries with backoff and jitter, circuit breakers, fallbacks, idempotency at the application layer. The failures LLMs add on top: semantically wrong output that parses fine, refusals of legitimate requests, format drift, and variable latency that scales with *output* length.
2. **The context window is a budget, not a bucket.** Allocate it explicitly like memory in an embedded system: system prompt + tools + history + retrieved docs + user input + *reserved output headroom*. Unbudgeted contexts fail at the worst time — the long conversation, the big document — usually by silently truncating the thing that mattered. And filling the window has a cost even when it fits: more input = more latency, more money, and more chance the model misses the relevant part.
3. **Latency, cost, and quality form a triangle you route across, not a knob you set once.** Different requests in the same product deserve different models. The design task is a router: classify request difficulty/stakes, send each tier to the cheapest model that meets the quality bar, and escalate on detected failure. A single "best model for everything" choice means you're overpaying on 80% of traffic or underserving the hard 20%.
4. **Perceived latency is time-to-first-token; real cost is dominated by tokens, and input≠output pricing.** Streaming doesn't make the model faster — it makes waiting feel like progress. Output tokens typically cost several× input tokens and dominate wall-clock time; the cheapest and fastest optimization is almost always "make the model say less" (tight output schemas, max_tokens caps, no restating the input).
5. **Non-determinism is a property, not a bug — handle it with contracts, not hope.** temperature 0 reduces variance but doesn't eliminate it (batching/floating-point effects). Anything downstream of a model must consume a *validated contract* (schema-checked JSON, enum'd fields) with a defined behavior when validation fails. Never let raw model text flow into code paths that assume structure.
6. **Prompts are code.** They live in version control, change via review, carry version identifiers into logs, get evaluated before deploy, and roll back like code. A prompt edited live in a dashboard with no eval run is a production hotfix without tests.

## Decision frameworks

**Model-tier routing:**
| Request profile | Route to | Reasoning |
|---|---|---|
| High-volume, low-stakes, well-templated (classify, extract, autocomplete, tag) | Smallest/cheapest tier, temp 0, few-shot | Quality bar is reachable by small models when the task is narrow; volume makes cost dominate |
| User-facing generation, moderate stakes (drafts, summaries, chat) | Mid tier, streaming on | Balance; users tolerate small imperfections but not 30s waits |
| Hard reasoning, high stakes, low volume (analysis, code gen, agent planning) | Top tier | Failure cost exceeds token cost by orders of magnitude |
| Detected failure at lower tier (validation error, refusal, low confidence) | Escalate one tier and retry once | Escalation-on-failure captures most of the quality of always-using-the-big-model at a fraction of cost |

Build the router interface first (`complete(request, tier)`), even if v1 routes everything to one model — retrofitting routing into direct SDK calls scattered across a codebase is painful.

**Streaming decision:** stream any user-visible generation longer than ~1–2 seconds; don't stream machine-consumed output (you can't validate a partial JSON, and buffering defeats the purpose — exception: streaming into a UI that renders progressively *and* re-validates on completion). When streaming structured-ish content, render progressively but keep a final-state pass that replaces the streamed render with the validated parse. Two UX details that separate good streaming from bad: render markdown incrementally without flicker (buffer to block boundaries), and make moderation/guardrail decisions *before* streaming starts or accept that you may need to retract visible text — retracting reads as creepy; prefer a pre-stream check on input plus post-hoc check that can append a correction.

**Latency budget arithmetic:** end-to-end latency ≈ network + queueing + prefill (scales with input tokens) + decode (scales with output tokens, dominates). Practical levers ranked by typical impact: (1) shorter outputs (schema, max_tokens, "no preamble"), (2) smaller model tier, (3) prompt-prefix caching (cuts prefill), (4) parallelizing independent calls, (5) speculative UX (start the likely call before the user finishes). Input trimming helps cost more than latency; output trimming helps both.

**Caching, two distinct kinds — don't conflate:**
- *Exact-response cache* (your infra): key = hash of (model, prompt-template-version, params, normalized input). Only for deterministic-intent calls (temp 0, no user-specific context). TTL by content volatility. Wins on: repeated classifications, popular queries, retries after downstream failures.
- *Prompt-prefix caching* (provider-side): providers discount and speed up requests whose prompt *prefix* matches a recent one. Consequence for prompt layout: **stable content first (system prompt, tool definitions, few-shot examples), variable content last (user input, retrieved docs, timestamp)**. One dynamic byte early in the prompt — a timestamp in the system prompt, a random request-ID, user name at the top — invalidates the entire prefix on every call. This regularly cuts input cost several-fold on multi-turn/agent workloads; check your provider's exact mechanics (some need cache markers, some are automatic).

**Long documents — chunk vs. long-context:**
| Situation | Choose | Because |
|---|---|---|
| Needle-finding / QA over one doc that fits in context | Whole doc in context, question *after* the doc, instruction restated at the end | Retrieval adds a failure stage you don't need; placement fights middle-of-context loss |
| Doc exceeds window, task is aggregation (summarize, extract all X) | Map-reduce: chunk → per-chunk extraction → merge pass | Parallelizable, each stage checkable; merge prompt needs dedup/conflict rules |
| Task needs cross-references far apart in a huge doc | Hierarchical: summarize sections, reason over summaries, drill into flagged sections | Pure chunking severs the long-range links the task depends on |
| Corpus of many docs, ad-hoc queries | RAG (see rag-systems skill) | Context stuffing doesn't scale past a handful of docs |
| Doc barely exceeds window | Try harder to shrink first: strip boilerplate/HTML/headers-footers, drop appendices | Cleaning is cheaper and less lossy than any splitting strategy |

**Guardrails — layer, in this order (each is optional, no layer is sufficient alone):**
1. *Input validation* (pre-model): length caps, format checks, cheap-classifier or rules for off-policy/abusive input, injection heuristics. Rejecting before the call is free.
2. *Structural output validation* (post-model): schema parse (Pydantic/JSON Schema), enum membership, ranges, ID-existence checks against your DB. On failure: one retry with the validation error appended to the prompt ("Your previous output failed validation: <error>. Output only corrected JSON."), then fallback.
3. *Semantic output checks*: rules first (forbidden claims, PII regexes, links to unknown domains), then optionally a cheap-model judge for policy questions rules can't express. Judges are noisy — use them to *flag*, not to silently rewrite.
4. *Human escalation*: define the trigger conditions up front (low confidence, high-stakes action class, repeated validation failure, user dispute) and the queue that receives them. A guardrail stack with no human tier just fails silently at the top.

## Failure modes and pitfalls

- **Retrying refusals identically.** A safety refusal is roughly deterministic — identical retry wastes money. Detect refusal shape (apology + decline, no requested structure) separately from API errors; handle by rephrasing/re-templating once, or degrading to a safe default plus escalation. Conversely DO retry (with backoff + jitter): 429, 5xx, timeouts. Distinguish the two paths explicitly in code.
- **No `max_tokens`/timeout bounds.** One pathological "explain in detail forever" output costs 50× median and blows your p99. Set max_tokens per call-site to ~2× expected output; set client timeouts that account for output length; kill and fall back rather than wait forever.
- **Counting tokens with the wrong tokenizer, or not at all.** `len(text)//4` drifts badly on code, CJK text, and long numbers; another model's tokenizer can be off by ±30%. Use the provider's counting endpoint or their official tokenizer package for budget decisions; measure, don't estimate, when the decision is "will this fit."
- **History management by "keep last N messages."** N messages can be 200 or 200,000 tokens. Budget by tokens; when over budget, summarize the oldest turns into a rolling summary rather than dropping them (dropped turns = the model re-asks answered questions; also: truncating mid-turn or orphaning a tool-result from its tool-call breaks many APIs). Pin the system prompt and any standing user facts outside the trimmable region.
- **Prompt-cache-hostile prompt layout** (dynamic content early). Symptom: cache hit rate near zero despite stable prompts. Audit the first divergent byte between consecutive requests' prompts; move everything before it that varies (timestamps are the classic offender — if the model needs "today's date", put it at the *end* or in the user turn).
- **Caching across prompt versions.** Response cache not keyed on template version serves stale-behavior outputs after a prompt deploy. Include prompt version and model ID in every cache key. Same bug in eval land: comparing eval runs across uncontrolled model updates when using a non-pinned model alias.
- **Validating structure but not referents.** JSON parses, but `product_id: "PRD-9931"` doesn't exist — the model invented a plausible ID. Every model-produced identifier, URL, price, or entity that your system will act on gets checked against ground truth before use. This is the most damaging production hallucination class because it *looks* valid.
- **One shared API-key/quota pool for interactive and batch traffic.** Nightly batch job exhausts rate limits; user-facing chat 429s. Separate keys/queues, run batch through the provider's batch API (typically ~50% cheaper) with client-side concurrency limits.
- **Treating provider errors as one bucket.** 400 context-length-exceeded needs trimming, not retry. 429 needs backoff (respect `retry-after`). 529/overloaded needs backoff + possible failover model. Content-filter errors need the refusal path. Map them explicitly; a generic `except: retry` loop turns a context-overflow into an infinite loop.
- **Shipping a prompt change without an eval run because "it's just wording."** Wording changes flip behavior. Minimum bar: rerun the regression set (see prompt-engineering skill), diff pass-rates, deploy behind a version flag so you can roll back per-version, and log the version with every request so incidents are attributable.
- **Degradation that's invisible to the user.** Fallback answers ("I couldn't process this fully; here's a partial summary") must be *labeled* as degraded, logged with the cause, and counted on a dashboard. Silent degradation looks like gradual quality decline nobody can localize.
- **Building conversation state on the provider.** Unless deliberately using a provider's state/session feature, the request should be reconstructible from your own store — you need that for replay, debugging, migration between providers, and evals over real traffic.
- **Mid-stream failures with no story.** Streams die at token 400 of 600: connection reset, content-filter stop, max_tokens hit. If the UI just freezes or shows the truncated text as if complete, users act on half an answer. Handle explicitly: detect the stop reason, show a "response interrupted — retry" affordance, and never persist a partial as final. For max_tokens stops on structured output, treat as validation failure (the JSON is guaranteed broken), not as content.
- **Retries wrapping non-idempotent side effects.** The model call itself is safe to retry; the *handler* around it may not be — "generate email then send" retried after a timeout that actually succeeded sends twice. Separate generation (retryable) from side effects (guarded by idempotency keys), and never put the side effect inside the retry loop.
- **Fallback model with different behavior, silently.** Failover from model A to model B keeps the service "up" while output format, refusal patterns, and quality shift under the same prompt — your validation was tuned for A. Every fallback pair needs its own eval run and possibly its own prompt variant; tag responses with the actual serving model so downstream metrics can be segmented.
- **Prompts and completions logged with PII into a system with different retention/access rules than the source data.** The chat log becomes a shadow copy of your most sensitive data, readable by every engineer with log access. Decide retention, redaction (at minimum: emails, phones, credit-card patterns before log write), and access control for LLM traffic logs as a first-class data store — and check the provider's own retention/training terms for your tier.
- **Unbounded client-side concurrency into rate limits.** A batch of 10,000 asyncio tasks all fire, hit 429s, all back off, all retry in sync (thundering herd). Use a semaphore sized to your rate limit, jittered backoff, and prefer the provider's batch API for anything not latency-sensitive.
- **No per-feature cost attribution.** One meter for the whole API key means the runaway feature is invisible until the invoice. Log tokens per request tagged by feature/prompt-version/tier from day one; set budget alerts per feature, not per account.

## Worked micro-examples

**1. Context budget for a doc-QA chat (128k window), written down before coding:**
```text
system + tools:           3,000   (stable → prompt-cache prefix)
few-shot examples:        2,000   (stable → still in prefix)
rolling history summary:  1,500
recent turns verbatim:    8,000   (token-budgeted, not message-counted)
retrieved chunks:        24,000   (top-k with cap; drop lowest-score first)
user input:               2,000   (hard cap, reject beyond with a clear error)
output reserve:           4,000   (max_tokens)
slack:                    ~5%
```
Total planned ≈ 45k of 128k — deliberately far under the window: cost and middle-loss scale with usage, not with the limit. Every region has an owner and an overflow rule; "retrieved chunks" shrinks first, "output reserve" never does.

**2. The call wrapper every LLM app needs (shape, in Python):**
```python
def robust_complete(req, tier="small", attempt=0):
    try:
        raw = client.complete(model=TIERS[tier], messages=req.messages,
                              max_tokens=req.max_tokens, temperature=0,
                              timeout=req.timeout_s)
    except RateLimited as e:
        sleep(backoff_jitter(attempt, retry_after=e.retry_after)); return robust_complete(req, tier, attempt+1)
    except ContextTooLong:
        return robust_complete(req.trimmed(), tier, attempt)      # trim, don't retry
    except (Overloaded, ServerError):
        if attempt < 3: sleep(backoff_jitter(attempt)); return robust_complete(req, tier, attempt+1)
        raise
    if looks_like_refusal(raw):                                    # refusal ≠ error ≠ bad JSON
        return handle_refusal(req)                                 # rephrase-once or degrade+escalate
    parsed = req.schema.validate(raw)                              # e.g. Pydantic
    if parsed.error:
        if attempt == 0:
            return robust_complete(req.with_repair_hint(parsed.error), tier, attempt+1)
        if tier != "large":
            return robust_complete(req, next_tier(tier), 0)        # escalate on persistent failure
        return degraded_fallback(req, cause="validation")
    log(req.prompt_version, tier, usage=raw.usage)                 # attribution for cost & incidents
    return parsed.value
```
The load-bearing details: three *distinct* failure paths (transport, refusal, validation), trim-not-retry on context overflow, escalation as the second retry, and prompt-version logging.

**3. Cost sanity check before building.** Feature: summarize support tickets, 50k/day, ~2k input + 300 output tokens each. At example rates of $0.25/M input, $1.25/M output (small tier): daily ≈ 50k × (2000×0.25 + 300×1.25)/1M = $43.75/day ≈ $1.3k/month. At a top-tier 20× price: ~$26k/month — that difference *is* the routing argument. If 90% route small and 10% escalate large: ≈ $3.9k/month. Prefix-caching the 800-token shared instructions cuts further. Do this arithmetic (with current real prices) in design review, not after the bill.

## Verification / self-check

- Does every model call site have: max_tokens, timeout, distinct handling for rate-limit vs refusal vs validation failure, and a defined fallback? Grep for naked `client.complete`/`messages.create` calls.
- Is the context budget written down, with an overflow rule per region and a fixed output reserve? Test empirically with an input at 100% and 120% of each cap.
- Are prompts version-controlled, is the version in every log line, and did the latest change run the regression set?
- Cache audit: is the response-cache key complete (model + prompt version + params + input)? Is the prompt prefix byte-stable across consecutive requests up to the intended split point?
- Kill-test the degradation paths in staging: block the API key, force 429s, inject malformed output — confirm users see labeled degradation, not stack traces or silence.
- Recompute the cost model against last week's real token logs; alert if $/request drifts >20% from plan.
