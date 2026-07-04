---
name: observability-engineering
description: Load when designing or reviewing instrumentation — logging, metrics, tracing, dashboards, alerts, SLOs — or when debugging why monitoring failed to catch an incident, why Prometheus/Datadog costs exploded, or when someone asks "what should we alert on" or "how do we know if this is healthy".
---

# Observability Engineering

## Core mental model

- **Instrument to answer questions you haven't thought of yet.** Monitoring answers known questions ("is CPU high?"); observability lets you ask new ones during an incident ("which customers on which endpoint see the errors, and what do those requests have in common?"). The test of instrumentation is not "do we have dashboards" but "can we go from a symptom alert to the differentiating dimension of the failure without shipping new code."
- **The three signals have different economics, and cost drives the decision rules.** Metrics: cost scales with *cardinality* (unique label combinations), NOT with traffic — cheap aggregates, useless for per-request questions. Logs: cost scales with traffic × verbosity — per-event detail, expensive at volume. Traces: logs with causality across services — the only signal that answers "where did the latency go in this request's life"; almost always sampled, so never a source of exact counts.
- **Alert on symptoms, page on user pain.** Users experience errors, latency, and wrong answers — never "CPU 90%". Cause-based alerts fire when nothing is wrong (CPU high, users fine) and stay silent in novel failures (CPU fine, users down). Cause signals belong on dashboards you consult *after* a symptom pages.
- **An SLO is a budget for imperfection that turns reliability into arithmetic.** 99.9% over 30 days = 43.2 minutes of full downtime, or 0.1% of requests failing continuously. Alerting thresholds, release gating, and reliability-vs-features prioritization all become "how fast are we spending the budget" — and that phrasing is what makes them principled instead of vibes.
- **Every signal you emit is a bill and a liability someone pays forever.** Unbounded label values, debug logs left at info, 100%-sampled traces, dashboards nobody owns — observability systems degrade by accretion, and they tend to fall over precisely during the traffic spikes when you need them. Default to fewer, wider, structured, owned.

## Decision frameworks

### Which signal for which question

| Question | Signal | Why |
|---|---|---|
| Is the service healthy right now? What do we alert on? | Metrics | Cheap, fast to query, math-friendly (rates, histograms, burn rates) |
| Why did *this specific* request fail? | Logs (joined by trace/request ID) | Full per-event detail, unsampled |
| Where in the call graph did the 3 seconds go? | Traces | The only signal with cross-service causality and per-hop timing |
| How many, exactly? (billing, compliance, "how many users hit this") | A counter metric or the source database — not traces (sampled), not logs (droppable) | Sampling and log-loss make the others estimates |
| Which users/tenants are affected? | Logs or trace attributes — NOT metric labels | Per-user labels explode metric cardinality; per-event signals absorb high cardinality natively |

Rule: emit a metric when you'll aggregate it, a log event when you'll inspect individual occurrences, a span when you'll follow a request across a boundary. Emitting the same fact as all three is usually waste — pick by the question it answers.

### Cardinality economics (why user-ID labels blow up Prometheus)

Each unique label-value combination is a separate time series held in memory (roughly 1–8KB each) and written to storage from then on. Series count *multiplies* across labels: `http_requests_total{path, status, method}` at 200 path templates × 10 statuses × 5 methods = 10,000 series — fine. Add `user_id` with 1M users → 10 *billion* potential series — Prometheus OOMs, or the Datadog bill does the equivalent to your budget.

Mechanical rules:
- Label values must come from a **small, closed, known-in-advance set**: status class, endpoint *template* (`/users/{id}` — never the raw path; raw URLs with embedded IDs are the most common accidental explosion), region, version, error *class*.
- Never label with: user/tenant/session/request IDs, emails, raw URLs, error *messages*, or container/pod IDs in high-churn autoscaling (every restart mints a fresh series set).
- Need per-user or per-tenant visibility? That's a logs/traces question, or a top-k sketch — not a metric label.
- Budget check before adding any label: multiply the metric's current series count by the new label's distinct values. If the product exceeds a few hundred thousand for one metric family, redesign.
- Histograms multiply too: one histogram = ~10–20 series per label combination (one per bucket). A histogram with a high-cardinality label is a double explosion.

### RED and USE — which to apply where

- **RED (Rate, Errors, Duration)** for every *service and endpoint*: requests/sec, error %, latency distribution. This is the user's view and the mandatory top row of every service dashboard.
- **USE (Utilization, Saturation, Errors)** for every *resource*: CPU, memory, disk I/O, network — and crucially the logical resources: connection pools, worker pools, semaphores, queue consumers. Saturation (queue depth, checkout wait time) is the leading indicator; utilization lags.
- Apply mechanically: list your services → RED each. List your resources, including hidden logical ones → USE each. "Latency is up but CPU is fine" is almost always saturation on a resource nobody put on a dashboard — the connection pool first among them.

### Structured logging design

- JSON (or logfmt) events; one wide event per request per service hop beats prose scattered across ten lines. One log with 30 fields can be grouped and filtered by anything; ten fragments must be joined by hand at 3am.
- Always include: `timestamp` (UTC, ISO-8601), `severity`, `service`, `version`/build, `trace_id` and `request_id` (the join keys to traces and across services), `duration_ms`, `status`, error class, plus the business dimensions you'll slice by (tenant, endpoint template, feature-flag state).
- Consistent field names and types across services: `user_id` everywhere — never `userId` in one service and `uid` in another; always the same type (mixed string/int breaks indexed search silently).
- Levels with teeth: ERROR = a human should act; WARN = would explain an incident in hindsight; INFO = state changes and request summaries; DEBUG = off in prod, toggleable at runtime without redeploy. If ERROR doesn't imply action, log-based alert fatigue begins.
- Never log: secrets/tokens/PII (a compliance incident laundered through the logging pipeline), unbounded payloads (one 10MB body at INFO can stall the pipeline), or anything inside a hot loop.

### Trace sampling strategy

- 100% tracing at scale is unaffordable; unbiased 1% sampling throws away 99% of the errors — which are the traces you wanted. Ladder:
  1. **Head sampling** (decide at request start; propagate the decision): simple and cheap. Keep 100% below ~100 req/s; 1–10% above.
  2. **Tail sampling** (decide after the trace completes, in the collector — e.g., the OpenTelemetry Collector's `tailsamplingprocessor`): keep 100% of errors and slow traces (> p99), ~1% of boring successes. The right default at scale — errors and outliers are precisely what traces exist for. Cost: the collector must buffer in-flight traces.
  3. Whatever you choose, the decision must be consistent across services (propagate `traceparent` and the sampling flag) — a trace with missing middle spans is worse than no trace.
- Never compute rates, ratios, or counts from sampled traces without weighting by the sample rate; better, use metrics for anything quantitative.

### Alert design rules

- Page only on: SLO burn (user-visible errors/latency/wrongness actively spending budget) or imminent hard failure (disk full in <4h, cert expiring, queue growing without bound). Everything else is a ticket or a dashboard panel.
- Every page must be **actionable** (a human can do something now), **urgent** (morning is too late), and **novel** (not a duplicate of a firing page). Fails any one → ticket, not page.
- Multi-window burn-rate alerting (the standard that works): page when burn rate ≥ 14.4× over 1h AND ≥ 14.4× over 5m (budget gone in ~2 days and still burning); page at ≥ 6× over 6h AND 30m; *ticket* at ≥ 1× over 3d. The short window arms fast and disarms after recovery; the long window filters blips.
- Symptom check for any proposed alert: "when this fires, is a user definitely having a bad time?" If the honest answer is "maybe", it's a dashboard line, not a page.
- Every alert carries an owner and a runbook link. Every page that ends in "no action taken" gets tuned or deleted within the week — that review loop, not the initial thresholds, is what keeps paging trustworthy.

### SLO / error-budget arithmetic (memorize the method, not numbers)

- Budget = (1 − SLO) × window. 99.9% over 30d → 0.001 × 43,200 min = **43.2 minutes**. 99.99% → 4.32 minutes — no human responds that fast, so 99.99% implies automated mitigation or it's fiction.
- Burn rate = observed error ratio ÷ (1 − SLO). At a 99.9% SLO, a 1% error rate is a 10× burn → budget exhausted in 3 days.
- Request-based budgeting: 60M requests/month at 99.9% = 60,000 failed requests to spend. A 30-minute incident at 5% errors and 4k req/min costs 6,000 requests — 10% of the monthly budget, in one line of arithmetic. This is how "can we ship the risky migration this week?" gets a numeric answer.
- Choose the SLI measurement point at the edge (load balancer), not the service's self-report — the service cannot see the requests that never reached it.
- Your SLO cannot exceed your serial hard dependencies': three chained 99.9% dependencies compound to ~99.7% before your own code fails once. Check the chain before promising the number.

## Failure modes & pitfalls

- **Averaged latency.** The mean hides everything — fast cache hits plus 10s timeouts average to "fine". Use histograms; read p50/p95/p99. And never average percentiles across instances or windows: p99s don't average. Aggregate the histograms first, then take the quantile — in PromQL, `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`; the `sum by (le)` *before* the quantile is the load-bearing part.
- **Alert on cause, silence on symptom.** A fleet of "CPU > 80%", "pod restarted", "replication lag" pages — and the actual outage (bad deploy returning 200 with wrong bodies) pages nobody, because every machine metric was green. Correction: the paging SLI must measure user-observable correctness at the point users enter; cause alerts get demoted to dashboard panels.
- **`rate()` window shorter than the scrape interval.** `rate(x[1m])` with a 60s scrape interval = an empty or gappy graph; use a range ≥ 4× the scrape interval. Related counter mistakes: applying `rate()` to a gauge (nonsense); graphing a raw counter and reporting "growth" (it's monotonic by construction — only its rate means anything); computing `max - min` by hand and getting burned by counter resets that `rate()` would have handled.
- **Cardinality explosion in one innocent deploy.** Someone adds a raw `path` (with UUIDs) or `customer_id` label; series count goes 50k → 40M; Prometheus OOMs — the monitoring dies *during* the incident it was needed for. Guards: PR review applies the closed-set rule to every new label; per-metric series limits in the ingestion pipeline; a standing top-10-metrics-by-series-count panel.
- **Sampled traces read as counts.** "Traces show only 12 errors" — at 1% sampling that's ~1,200. Always surface the sample rate next to any trace-derived number; route counting questions to counters.
- **The dashboard graveyard.** Hundreds of auto-generated and incident-specific dashboards; during an outage nobody knows which one is current, and half the panels query metrics that no longer exist. Corrections: one owned, curated dashboard per service — RED top row, USE below, deploy-event annotations on every time panel; per-incident scratch dashboards get dated and deleted; anything unviewed for 90 days is auto-archived. A dashboard nobody can interpret under pressure has negative value: it burns incident minutes.
- **Ten narrow log lines instead of one wide event.** `"entering handler"`, `"got user"`, `"calling payment"` — no shared ID, no duration, six write amplifications per request. Consolidate into one structured event per request per hop, plus a trace for the cross-service view. Costs less, answers more.
- **Broken trace-context propagation.** Service A traces, service B logs, and nothing joins them — the `traceparent` header dies at a proxy or a message queue, or B's logs omit `trace_id`. The triad's whole value is the join keys; instrument propagation first (OpenTelemetry auto-instrumentation covers HTTP/gRPC boundaries; queues usually need manual context injection), enrich fields second.
- **Health endpoint standing in for an SLI.** `/healthz` returns 200 (process up) while the service returns 100% errors to users (dependency down; the handler doesn't check it). Liveness ≠ readiness ≠ user experience. The paging signal must be derived from real traffic, not from a synthetic self-report.
- **Alert fatigue treated as an on-call attitude problem.** More than ~2 pages per shift, or >30% of pages ending in "no action", and humans *will* start ignoring pages — then the real one gets slept through. This is an alert-quality bug with a metric: track pages/shift and actionable-fraction weekly, and delete or ticket-ify the offenders with the same rigor as error budgets.
- **Postmortem-driven metric accretion.** Every incident adds the one cause-metric that would have caught *that* incident; three years later there are 400 alerts and the next novel failure still pages nobody. The durable fix is almost always better symptom SLI coverage plus richer wide events (more dimensions on existing logs/traces so novel questions are answerable), not another special-case alert.
- **Observability coupled to the failure domain.** Metrics and logs shipped through the same cluster/region/queue that's failing → blind exactly when it matters. Keep the telemetry path out-of-band: separate cluster or vendor, local buffering on agents, and an external black-box probe that doesn't trust your infrastructure at all.

## Worked micro-example

**Alerting design for a checkout API: 2M requests/day (~60M/month), SLO = 99.9% availability and 99% of requests under 500ms, 30-day window.**

Budget: allowed failures = 0.001 × 60M = **60,000 requests/month**. Latency budget: 0.01 × 60M = 600,000 slow requests.

SLI, measured at the load balancer: `good = status < 500 AND duration < 500ms`; `SLI = good / total`. (5xx emitted by the LB itself counts even when the pods think they're fine — that is the point of measuring at the edge.)

Fast-burn page (availability side), PromQL sketch:

```promql
(
  sum(rate(lb_requests_total{status=~"5.."}[5m]))
    / sum(rate(lb_requests_total[5m])) > 14.4 * 0.001
) and (
  sum(rate(lb_requests_total{status=~"5.."}[1h]))
    / sum(rate(lb_requests_total[1h])) > 14.4 * 0.001
)
```

Check the arithmetic: 14.4× burn = 1.44% error rate; 1.44% of 2M/day = 28,800 errors/day → the 60k budget is gone in ~2.1 days — worth waking someone. The 5m window arms the alert quickly and disarms it after recovery; the 1h window keeps blips from paging. Verify the blip case: a 90-second total outage at 1,400 req/min costs ~2,100 requests = 3.5% of the monthly budget — real money, but the 1h-window burn is only ~2.5% error rate-hours, far below threshold: no page, correctly; it becomes a ticket via the 3-day 1× window. Add the 6×/(6h AND 30m) pair as a slower page. Pod restarts, CPU, GC pauses, dependency latency: dashboard panels with deploy annotations, consulted after the page — because the first incident question is always "what changed".

## Verification / self-check

- For each proposed alert: does it fire only when users are hurting? Is there a runbook? Compute the "budget exhausted in X days" number behind the threshold explicitly — if you can't, the threshold is a guess.
- For each metric label: is its value set closed and enumerable *today*? Multiply out the series count including histogram buckets.
- For every percentile: was aggregation histogram-first? Any averaged percentile invalidates the number — recompute.
- For any count derived from traces or sampled logs: state the sample rate and the weighting, or move the claim to a counter.
- Run the incident drill mentally: symptom page fires → which dashboard, which query, which log filter takes you to the differentiating dimension in under 5 minutes? If any hop needs a code change, the instrumentation isn't done.
- Check the SLO against serial dependencies (multiply their SLOs) and against the measurement point (edge, not self-report). Check the telemetry path survives the failure it must observe.
