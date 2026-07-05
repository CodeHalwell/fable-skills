---
name: observability-engineering
description: Load when designing or reviewing instrumentation — logging, metrics, tracing, dashboards, alerts, SLOs — or when debugging why monitoring failed to catch an incident, why Prometheus/Datadog costs exploded, or when someone asks "what should we alert on" or "how do we know if this is healthy".
---

# Observability Engineering

Compact checklist. The standard expert material — cardinality economics, symptom-vs-cause paging, multi-window burn rates (14.4×/1h+5m, 6×/6h+30m, ticket at 1×/3d), error-budget arithmetic, `sum by (le)` before `histogram_quantile`, rate-window ≥ 4× scrape interval, tail sampling in the OTel Collector, wide events, serial-dependency SLO ceilings, edge-measured SLIs, RED/USE with pool saturation as the missed resource, healthz-is-not-an-SLI — is assumed known and appears only as anchors.

## Anchors (apply without re-derivation)

- Metric labels: closed, enumerable-today sets only (endpoint *template*, status class, region, version, error *class*). Never IDs, raw paths/URLs, messages, or high-churn pod names. Budget check before any new label: current series × new distinct values; >few hundred thousand per family = redesign. Histograms multiply by ~10–20 buckets.
- Page only on SLO burn or imminent hard failure; every page actionable + urgent + novel; runbook link; multi-window burn-rate pairs, short window arms/disarms fast.
- Budget = (1−SLO) × window: 99.9%/30d = 43.2 min or 60k of 60M requests. Burn rate = error ratio ÷ (1−SLO). 99.99% = 4.32 min/month = automated mitigation or fiction. Serial 99.9% deps: three chained ≈ 99.7% ceiling.
- Percentiles never average — aggregate histograms first; `histogram_quantile(0.99, sum by (le) (rate(..._bucket[5m])))`.
- Counters: `rate()` before `sum()` (reset handling), never `rate()` a gauge, range ≥ 4× scrape interval.
- Traces answer "where did the latency go"; never counts (state the sample rate next to any trace-derived number). Tail-sample: 100% of errors + >p99, ~1% of successes; propagate `traceparent` + decision consistently or the trace is worse than none; queues need manual context injection.
- Logs: one wide structured event per request per hop; consistent field names/types across services (`user_id` everywhere, same type); `trace_id` as the join key; ERROR means a human acts; no secrets/PII/unbounded payloads.
- SLI at the load balancer, not self-report — the service can't see requests that never arrived.

## Corrections beyond the standard answers (the delta)

- **Postmortem-driven metric accretion is the failure mode of "learning from incidents."** Every incident adds the one cause-alert that would have caught *that* incident; three years later there are 400 alerts and the next novel failure still pages nobody — cause-alert coverage of past incidents converges to zero coverage of future ones. The durable fix after an incident is almost always (a) better symptom-SLI coverage at the edge and (b) *more dimensions on existing wide events* so novel questions are answerable — not another special-case alert. In postmortem review, challenge any action item that adds a cause alert.
- **Alert quality is a measured control loop, not an attitude.** Track two numbers weekly: pages/shift and fraction of pages ending "no action taken." More than ~2 pages/shift or >30% no-action means humans have already started ignoring pages — the real incident gets slept through. Every no-action page gets tuned, ticket-ified, or deleted within the week. Without this loop the initial thresholds don't matter.
- **The telemetry path must not share fate with the failure it observes.** Metrics/logs shipped through the same cluster, region, or queue that is failing go blind exactly when needed. Separate cluster or vendor, local agent buffering, plus one external black-box probe that trusts nothing of your infrastructure — and a deadman alert (`absent()`) proving the pipeline itself is alive.
- **Dashboards: one owned, curated board per service** (RED top row, USE below, deploy annotations on every time panel — the first incident question is "what changed"). Dated scratch dashboards for incidents, deleted after; auto-archive anything unviewed 90 days. A dashboard nobody can interpret under pressure has negative value: it burns incident minutes.

## Verification / self-check

- Per alert: fires only when users hurt? runbook? compute "budget gone in X days" behind the threshold — no number, no threshold.
- Per label: closed set today? multiply out series incl. buckets.
- Per percentile: histogram-first aggregation, or invalid. Per trace-derived count: sample rate stated, or move to a counter.
- Incident drill: symptom page → which dashboard, query, log filter reaches the differentiating dimension in <5 min without a code change?
- SLO vs serial dependencies multiplied; SLI at the edge; telemetry path survives the failure it must observe.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 12 baseline (compressed to anchors), 0 partial, 0 delta.
- Opus cold reproduced: full cardinality math with per-series RAM, exact burn-rate table, budget arithmetic incl. the 10%-of-budget incident example, histogram-first quantiles, 4× rate-window rule, OTel tail-sampling, wide events, 99.7% serial ceiling, edge SLIs, pool-saturation as the missed USE resource.
- Kept expanded (unprobed gaps in Opus's pitfall coverage): postmortem metric accretion (cause-alert convergence to zero future coverage), the quantified alert-review loop (2 pages/shift, 30% no-action), fate-shared telemetry, dashboard curation discipline.
