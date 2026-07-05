---
name: estimation-and-capacity-planning
description: Load when sizing a system before building it or scaling it — back-of-envelope estimates, "how many servers/how much storage/what instance size" questions, throughput and concurrency math, load-test design and interpretation, headroom and utilization targets, growth runway planning, or database capacity sizing.
---

# Estimation & Capacity Planning

## Core mental model

- **Estimate before you architect; the envelope decides the architecture.** Half the time the 15-minute envelope dissolves the hard problem ("one Postgres, no distributed anything") — that conclusion is the deliverable, and it's worth stating as the deliverable.
- **Little's Law (`L = λ × W`) at every pooled resource** — in-flight requests, DB connections, worker slots, queue drain ETAs. No pool sized by folklore.
- **Utilization and latency are enemies** (M/M/1 wait ≈ service × ρ/(1−ρ): 1× at 50%, 4× at 80%, 9× at 90%, 19× at 95%). Latency tiers target 60–75% at peak; batch tiers 85–95%. The table is the *optimistic* bound — burstiness and heavy-tailed service times push real queues above it.
- **Produce three numbers, always:** average / daily peak (×3–5 for consumer diurnal) / event peak. Designs quoting one number are hiding the ratio — the classic 46× error is sizing to the average.
- **Order of magnitude is the goal.** Round hard (1 day ≈ 10⁵ s; 1M/day ≈ 11.6/s; 1KB × 1M/day ≈ 1GB/day ≈ 365GB/yr, ×2–3 overhead; telemetry dwarfs business data ~10:1 — estimate it separately or the storage number is fiction). Cross-check every estimate by a second independent path; disagreements are where the findings live.

## Hard limits an envelope must check

| Limit | Typical default | Where it bites |
|---|---|---|
| Postgres/MySQL connections | ~100–500 | The *first* limit apps hit — before QPS or CPU |
| File descriptors / process | 1k–65k | Proxies, WebSocket servers |
| Ephemeral ports per (src,dst) | ~64k | NAT/SNAT-heavy outbound services |
| LB/gateway integration timeout | ~29–60s | Long requests must become async |
| Cloud quotas (instances, Lambda concurrency, API rates) | per-account | Discovered during the event you planned for — audit when the plan is written, raised in days |
| Kafka/Kinesis per-partition throughput | ~1–10s MB/s | Ordered streams don't scale past the partition |

## Latency budgets

Budget with dependencies' **p99s, not p50s** — a parallel fan-out completes at the max of N draws, so overall p95 across 5 dependencies needs each at ~p99 (0.95^(1/5) ≈ 0.99); sum for sequential, max for parallel. Count sequential round-trips first: collapsing 4 calls to 2 or batching N lookups beats any compute optimization by an order of magnitude.

## Reading a running system in ten minutes

When the system exists, estimate from telemetry, not first principles: peak ratio from 30-day max-of-1-min ÷ average; concurrency from RPS × p50 vs pool sizes; working set from the cache-hit-rate plateau; growth from 90-day fits *checking the second derivative*; failure headroom from yesterday's deploy or AZ rebalance as a natural experiment.

## Growth and runway

Viral/consumer growth plans in doublings: "what breaks at 2×/4×/8×?" — each answer usually differs (2×: connections; 4×: write IOPS; 8×: the single-region design). That breakpoint list *is* the capacity plan. Alert on **projected time-to-exhaustion** (< 2× the lead time of the fix), never percent-full — 40%-full growing 5%/week is an incident in six weeks; 80%-full growing 1%/yr is fine. Check growth for compounding: 95GB/mo accelerating 10%/mo turns a naive 22-month runway into 14.

## Headroom policy

State it per tier with the reason: queueing (the 80→95% region is a 5× latency cliff), zone loss (N zones at 80% = 120% on N−1; target ≤ (N−1)/N × latency-safe), autoscaling lag (headroom buys the 1–5 min), and variance (70% must be p95-at-peak of fine-grained samples — minute-averages hide 10-second saturation bursts). **Autoscaling arithmetic before HPA/ASG configs:** absorbable spike = headroom ÷ (slope × lag); if traffic doubles in 2 min and lag is 4 min, no reactive policy works — the options are permanent headroom, pre-scaling on schedule/signal, or admission control converting the spike into delay. Budget the retry storm riding on top of event peak (~2×), with load shedding and jittered backoff in the overload plan.

## Load testing that predicts reality

- **Coordinated omission invalidates closed-loop tools** (fixed-thread JMeter-style): the tool slows down during the stall, dropping exactly the samples that show it — reported p99 can be off by 10–1000×. Open-loop constant-arrival tools (wrk2, Vegeta, k6 arrival-rate, Gatling open profiles) or corrected modes only.
- Test the four shapes: ramp (the knee where latency departs linearity *is* your capacity — not first-error, and never extrapolate linearly through it), spike (autoscaling lag, cold paths), soak (hours: leaks, fd exhaustion, log fill, pool decay — every "passed load test, died Saturday" is a missing soak), overload (shed or collapse?).
- Realism: production-scale data (100 rows lie about 100M-row query plans), Zipf keys (uniform overestimates hit rate and underestimates hot-key contention), warmed steady state, and the production entry path (CDN+WAF+LB+TLS) — not a generator inside the VPC hitting the service directly.
- The deliverable is a sentence: "knee at 3,200 req/s on 4×c7g.xlarge, p99 180ms, linear to the knee, sheds gracefully to 2,800" — "handled 5k req/s" without tail latency reports nothing.

## Database sizing

Four independent axes — the binding one is usually not the one being discussed: working set vs RAM (the falls-out-of-memory cliff is brutal; estimate hot set, not total), write IOPS (WAL + data + every index, × replication), **connections** (Little's Law from app concurrency; pooler mandatory beyond a few hundred — each Postgres connection is a process), storage growth (×2–3). Read scaling order: cache → replicas (mind read-your-writes lag) → sharding last, priced in engineer-years; the envelope usually shows you're a decade from needing it.

**Cache sizing from the distribution, not vibes:** under Zipf, top ~1% of keys ≈ 50–70% of traffic, top ~10% ≈ 90% — so 100M × 2KB items at a 90% target ≈ cache ~10M × ~2.1KB ≈ 21GB. Then check the *other two* constraints: **bandwidth** (20k/s × 20KB objects saturates a 10Gbps NIC long before memory matters — caches go bandwidth-bound at large values) and **miss-path capacity** (10% miss QPS at steady state, 100% at cold start — stampede protection or the first cache restart is an outage).

## How an expert thinks through this

*"Notification service: 8M users × 3/day, push + inbox, must handle a Super Bowl moment."*

Baseline ≈ 280/s avg, ~1k/s peak — tiny. The blast is the design load: 8M in 10 min ≈ 13k/s — but it's a *batch disguised as a spike*; nobody perceives a push 4 minutes into the window, so the queue absorbs it — reject 13× standing capacity. Workers by Little's Law: 13k × 0.05s = 650 concurrent ÷ 50/worker ≈ 13, run 20; pre-scale before scheduled blasts (spike onset in seconds, scale-out in minutes — the arithmetic says reactive loses). The *actual* hard part is 13k inbox rows/s on the primary: batched inserts (10–50× cheaper/row) + daily partitions (retention = partition drop), fan-out-on-read noted as the 5× escape hatch. Storage forces retention policy at launch (~11TB/yr raw×2.5 → 90 days caps at 2.7TB). Check the platform quota: if FCM/APNS caps below 13k/s, the window stretches — a product conversation, not an engineering one. Sanity-check by a second path (8M ÷ 13 workers ÷ 50 ÷ 20/s ≈ 10.2 min ✓). Deliver: three load numbers, breakpoint list, runway alerts.

## Presenting an estimate

Lead with the decision ("one Postgres through ~10×"), then the two lines of math. Ranges with the driver named ("2.7–11TB/yr *depending on retention*") turn scary intervals into decisions the audience owns. Never show one load number (executives anchor on it). State the first breakpoint and its fix lead time — that sentence is capacity *planning* vs capacity reporting. Attach falsifiable assumptions ("20 req/user/day — analytics, March").

## Failure modes (checklist)

- Sizing to the average; one-number designs.
- Closed-loop tail latencies believed; averages hiding 10s saturation bursts.
- Connection math skipped (400 instances × pool 20 = 8,000 vs `max_connections` 500); pooler before `max_connections` bumps.
- Cold caches / toy data / uniform keys; missing soak; extrapolating through the knee.
- Utilization "efficiency" targets on latency tiers; no zone-loss utilization story.
- Percent-full alerts; retries missing from peak math; budgets from p50s; test path ≠ production path; quota audit missing; the founding envelope never re-run at each order of magnitude (conclusions flip at predictable thresholds).

## Verification / self-check

1. Arithmetic shown, assumptions inline, re-derivable by the reader.
2. Second independent path agrees within 2–3×.
3. Three load numbers + breakpoint list at 2×/4×/8×.
4. Little's Law applied at each pool.
5. Headroom policy stated per tier with its reason.
6. Tail-latency claims traced to open-loop measurement.
Stopping rule: done when the *decision* is insensitive to remaining uncertainty — if the answer is "one Postgres" at 300/s or 900/s, stop refining and go measure the real system.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails Little's Law applications, the M/M/1 table, coordinated omission (with the canonical 10–1000× error), N−1 zone math, time-to-exhaustion alerting, autoscaling-lag arithmetic, telemetry-vs-business-data ratios, and p99 fan-out budgeting — all compressed to anchors.
- The one sharpening: cache sizing's *secondary* constraints — Opus checks miss-path/stampede capacity but not NIC bandwidth saturation at large value sizes; both now stated. This skill's residual value is the worked discipline (three numbers, breakpoint lists, second-path cross-checks) as a review scaffold, not facts Opus lacks.
