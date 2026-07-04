---
name: observability-engineering
description: Load when designing or reviewing instrumentation — logging, metrics, tracing, dashboards, alerts, SLOs — or when debugging why monitoring failed to catch an incident, why Prometheus/Datadog costs exploded, or when someone asks "what should we alert on" or "how do we know if this is healthy".
---

# Observability Engineering

## Core mental model

- **Instrument to answer questions you haven't thought of yet.** Monitoring answers known questions ("is CPU high?"); observability lets you ask new ones during an incident ("which customers on which endpoint see the errors, and what do those requests share?"). The test of instrumentation is not "do we have dashboards" but "can we go from a symptom alert to the differentiating dimension of the failure without shipping new code."
- **The three signals have different economics, and cost drives the decision rules.** Metrics: cost scales with *cardinality* (unique label combinations), NOT traffic — cheap aggregates, useless for per-request questions. Logs: cost scales with traffic × verbosity — per-event detail, expensive at volume. Traces: logs with causality across services — the only signal that answers "where did the latency go in this request's life"; almost always sampled, so never a source of exact counts.
- **Alert on symptoms, page on user pain.** Users experience errors, latency, and wrongness — never "CPU 90%". Cause-based alerts fire when nothing is wrong (CPU high, users fine) and stay silent in novel failures (CPU fine, users down). Cause signals belong on dashboards you consult *after* a symptom pages.
- **An SLO is a budget for imperfection that converts reliability into arithmetic.** 99.9% over 30 days = 43.2 minutes of full downtime, or 0.1% of requests failing continuously. Every alerting, release, and prioritization decision can be phrased as "how fast are we spending the budget" — and that phrasing is what makes alert thresholds principled instead of vibes.
- **Every signal you emit is a bill and a liability someone pays forever.** Unbounded label values, debug logs left at info, 100%-sampled traces, dashboards nobody owns — observability systems degrade by accretion. Default to less, structured, and owned.

## Decision frameworks

**Which signal for which question:**

| Question | Signal | Why |
|---|---|---|
| Is the service healthy right now? Alert on what? | Metrics | Cheap, fast to query, math-friendly (rates, percentiles, burn rates) |
| Why did *this specific* request fail? | Logs (with trace/request ID) | Full per-event detail |
| Where in the call graph did the 3s go? | Traces | Only signal with cross-service causality and timing |
| How many exactly? (billing, compliance) | Neither traces (sampled) nor logs (droppable) — a counter metric or the source DB | Sampling and log-loss make the others estimates |
| Which users/tenants are affected? | Logs or trace tags — NOT metric labels | Per-user in metrics = cardinality explosion; per-event signals absorb high cardinality natively |

Rule: emit a metric when you'll aggregate it; a log when you'll inspect one; a span when you'll follow a request across a boundary. Emitting the same fact as all three is usually waste — pick by the question.

**Cardinality economics (why user-ID labels blow up Prometheus).** Each unique label-value combination creates a separate time series held in RAM (~1–8KB each) and written forever after. Series count multiplies across labels: `http_requests_total{path, status, method}` with 200 paths × 10 statuses × 5 methods = 10,000 series — fine. Add `user_id` with 1M users and it's 10 *billion* potential series — Prometheus (or your Datadog bill) dies. Mechanical rules:
- Label values must come from a **small, closed, known-in-advance set**: status class, endpoint *template* (`/users/{id}`, never the raw path — raw URLs with IDs are the most common accidental explosion), region, version.
- Never label with: user/tenant/session/request IDs, email, raw URL, container ID in high-churn autoscaling (each pod restart = new series set), error *messages* (use error *class*).
- Need per-user visibility? That's a logs/traces question, or a top-k sketch — not a metric label.
- Budget check before adding a label: multiply current series count by the new label's distinct values; if the product exceeds ~a few 100k for one metric, redesign.

**RED and USE — which to apply where.** RED (Rate, Errors, Duration) for every *service/endpoint*: requests/sec, error %, latency distribution — this is the user's view and the template for every service dashboard's top row. USE (Utilization, Saturation, Errors) for every *resource*: CPU, memory, disk I/O, connection pools, queues — where saturation (queue depth, wait time) is the leading indicator; utilization lags. Apply mechanically: list your services → RED each; list your resources (including logical ones: pools, semaphores, queue consumers) → USE each. Saturation on a hidden resource (connection pool!) explains most "latency is up but CPU is fine" mysteries.

**Structured logging design:**
- JSON (or key-value) events, one per request per service hop, not prose scattered across ten lines. Wide events beat many narrow ones: one log with 30 fields lets you group/filter by anything; ten fragments require joining by hand.
- Always include: `timestamp` (UTC, ISO-8601), `severity`, `service`, `version`, `trace_id`/`request_id` (the join key to traces and across services), `duration_ms`, `status`, error class, and the business dimensions you'll slice by (tenant, endpoint template, feature flag state).
- Consistent field names and types across services (`user_id` everywhere, never `userId` in one and `uid` in another; always a string or always an int — mixed types break indexing).
- Levels with teeth: ERROR = someone should act; WARN = would explain an incident later; INFO = state changes and request summaries; DEBUG = off in prod, toggleable at runtime. If ERROR doesn't imply action, alert fatigue starts in the logs.
- Never log: secrets/tokens/PII (compliance incident via the logging pipeline), unbounded payloads (one 10MB body at info = pipeline outage), inside hot loops.

**Trace sampling strategy.** 100% tracing at scale is unaffordable; unbiased 1% sampling discards 99% of the errors you care about. Ladder:
1. Head sampling (decide at request start, propagate the decision): cheap, simple; keep 100% below ~100 req/s, 1–10% above.
2. **Tail sampling** (decide after the trace completes, in the collector — e.g., OpenTelemetry Collector `tailsamplingprocessor`): keep 100% of errors and slow traces (>p99), 1% of boring successes. This is the right default at scale — errors and outliers are precisely what traces are *for*.
3. Whatever you sample, keep the sample *decision* consistent across services (propagate context) — a trace with missing middle spans is worse than no trace.
- Corollary: never compute rates or percentages from sampled traces without weighting by sample rate; prefer metrics for those.

**Alert design rules:**
- Page only on: SLO burn (user-visible errors/latency/wrongness) actively spending budget, or imminent hard failure (disk full in <4h, cert expiry, queue growing unboundedly). Everything else is a ticket or a dashboard.
- Every page must be: **actionable** (a human can do something now), **urgent** (waiting until morning makes it worse), and **novel** (not a duplicate of an existing page). Fails any → ticket, not page.
- Multi-window burn-rate alerting (the standard that works): page when burn rate ≥ 14.4× over 1h AND ≥ 14.4× over 5m (budget gone in ~2 days, still burning); page at ≥ 6× over 6h/30m; *ticket* at ≥ 1× over 3d. The short window confirms it's still happening (stops paging after recovery); the long window filters blips.
- Symptom check for any proposed alert: "when this fires, is a user definitely having a bad time?" If the honest answer is "maybe", it's a dashboard line.
- Every alert needs an owner and a runbook link; every page that results in "no action taken" gets tuned or deleted that week — that review loop, not the initial thresholds, is what keeps paging trustworthy.

**SLO / error-budget arithmetic (memorize the method, not the numbers):**
- Budget = (1 − SLO) × window. 99.9% / 30d → 0.001 × 43,200 min = **43.2 min**. 99.99% → 4.32 min (no human responds that fast — 99.99% implies automated mitigation or it's fiction).
- Burn rate = (observed error ratio) / (1 − SLO). At 99.9% SLO, a 1% error rate = 10× burn → budget gone in 3 days.
- Request-based budgeting: 100M requests/month at 99.9% = 100k failed requests to "spend" — a 30-min 5%-error incident at 4k req/min costs 6k requests = 6% of the monthly budget. This arithmetic is how you answer "can we ship the risky migration this week?" with a number.
- Choose the SLO from user tolerance and the *measurement point* (load balancer, not the service's own view — the service can't see requests that never arrived), and remember your SLO can't exceed your hard dependencies' — a 99.9% SLO atop a serial chain of three 99.9% dependencies is already spent (~99.7% compound) before your own code fails once.

## Failure modes & pitfalls

- **Averaged latency.** Mean latency hides everything (bimodal: fast cache hits + 10s timeouts average to "fine"). Use histograms; look at p50/p95/p99. And never average percentiles across instances or windows — p99s don't average; aggregate the histograms first, then take the percentile (`histogram_quantile(0.99, sum by (le) (rate(...)))` in PromQL — the `sum by (le)` before the quantile is the load-bearing part).
- **Alert on cause, silence on symptom.** Fleet of "CPU > 80%", "pod restarted", "replication lag" pages, yet the outage (bad deploy returning 200-with-wrong-body) pages nobody. Correction: the SLI must measure user-observable correctness where users enter; cause alerts become dashboard panels.
- **`rate()` window shorter than scrape interval / counter mistakes.** `rate(x[1m])` with 60s scrape = empty graph. Use ≥ 4× scrape interval. Related: applying `rate()` to a gauge (nonsense), or graphing a raw counter and "seeing growth" (it's monotonic by definition; only its rate means anything). Counter resets on restart are handled by `rate()` — but not by naive `max - min` queries.
- **Cardinality explosion in one deploy.** Someone adds `path` (raw, with UUIDs) or `customer_id` as a label; series count goes 50k → 40M; Prometheus OOMs — the monitoring dies *during* the incident it was needed for. Guard: PR review checks every new label against the closed-set rule; per-metric series limits in the pipeline.
- **Sampled traces used as counters.** "Traces show only 12 errors" — at 1% sampling that's ~1,200. Always display and reason with sample-rate weighting; use metrics for counts.
- **The dashboard graveyard.** Hundreds of auto-generated or incident-specific dashboards; during an outage nobody knows which is current, and half the panels query dead metrics. Corrections: one owned, curated dashboard per service with the RED top row and USE below; per-incident scratch dashboards get deleted or dated; anything not viewed in 90 days is archived automatically. A dashboard nobody can interpret under pressure is negative value — it burns incident minutes.
- **Logging the same request at 6 points as prose.** `"entering handler"`, `"got user"`, `"calling payment"`, ... none with a shared ID or duration. Ten narrow log lines cost more and answer less than one wide structured event per request plus a trace. Consolidate.
- **Missing trace context propagation.** Service A traces, service B logs, and nothing joins them — because the `traceparent` header dies at B, or B's logs omit `trace_id`. The whole point of the triad is the join keys; instrument propagation first (OpenTelemetry auto-instrumentation gets HTTP/gRPC boundaries for free), fields second.
- **Health = "the process responds".** `/healthz` returns 200 while the service serves 100% errors (dependency down, but the handler doesn't check it). Liveness ≠ readiness ≠ actual SLI. Never let a health endpoint stand in for a symptom SLI.
- **Alert fatigue treated as an on-call attitude problem.** If >~2 pages per shift or >30% of pages end in "no action", people *will* start ignoring pages, and the real one gets slept through. This is an alert-quality bug: delete, ticket-ify, or re-threshold — measured weekly, with the same rigor as error budgets.
- **Instrumenting after the incident, forever.** Each postmortem adds the one metric that would have caught *that* incident — accreting cause-alerts. The durable fix is almost always: better symptom SLI coverage + richer wide events (more dimensions on existing logs/traces), not another special-case metric.

## Worked micro-example

**Designing alerting for a checkout API, 2M requests/day, SLO 99.9% availability + 99% of requests < 500ms, 30-day window.**

Budget: errors allowed = 0.001 × 60M = 60,000 failed requests/month. Latency budget: 1% of 60M = 600k slow requests.

SLI (measured at the load balancer): `good = requests with status < 500 AND duration < 500ms`; `SLI = good/total` per window. (5xx from the LB counts even when the pods think they're fine — that's the point of measuring at the edge.)

Burn-rate pages (PromQL sketch, availability side):
```promql
# fast burn: budget gone in ~2 days — page immediately
(
  sum(rate(lb_requests_total{status=~"5.."}[5m])) / sum(rate(lb_requests_total[5m])) > 14.4 * 0.001
) and (
  sum(rate(lb_requests_total{status=~"5.."}[1h])) / sum(rate(lb_requests_total[1h])) > 14.4 * 0.001
)
```
14.4× burn = 1.44% error rate. Check the math: 1.44% of the daily 2M = 28.8k errors/day → 60k budget gone in ~2.08 days. The 5m window arms the alert quickly and disarms it after recovery; the 1h window stops a 90-second blip from paging (a 90s 100%-outage = 100×0.025h burn ≈ 2.5% of the hourly threshold — no page, correctly: it cost only ~2,100 requests, 3.5% of the monthly budget, which is a ticket-and-postmortem, not a 3am page). Add the 6×/6h+30m pair as a slower page and 1×/3d as a ticket. Everything else — pod restarts, CPU, GC pauses, dependency latency — goes on the dashboard consulted after the page, tagged with `deploy` annotations, because the first question is always "what changed".

## Verification / self-check

- For each proposed alert: does it fire only when users are hurting, does the runbook say what to do, and what's the burn-rate math behind the threshold? Compute the "budget exhausted in X" number explicitly.
- For each metric label: is its value set closed and enumerable now? Multiply out the series count.
- For percentile claims: was the aggregation histogram-first? Any averaged percentile invalidates the number.
- For any count derived from traces or sampled logs: state the sample rate and weight, or move the claim to a counter metric.
- Run the incident drill mentally: symptom page fires → which dashboard, which query, which log filter gets you to the differentiating dimension in <5 minutes? If any hop requires a code change, the instrumentation design isn't done.
- Check the SLO against dependencies' SLOs (serial chain multiplies) and against the measurement point (edge, not self-reported).
