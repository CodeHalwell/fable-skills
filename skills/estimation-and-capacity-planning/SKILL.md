---
name: estimation-and-capacity-planning
description: Load when sizing a system before building it or scaling it — back-of-envelope estimates, "how many servers/how much storage/what instance size" questions, throughput and concurrency math, load-test design and interpretation, headroom and utilization targets, growth runway planning, or database capacity sizing.
---

# Estimation & Capacity Planning

## Core mental model

- **Estimate before you architect; the envelope decides the architecture.** 100 requests/sec fits on two boring VMs with Postgres — most "scalability" designs for it are waste. 100k req/s is a genuinely different system. The 15-minute back-of-envelope is what tells you which problem you actually have, and half the time it dissolves the hard problem entirely.
- **Little's Law is the workhorse:** `L = λ × W` — concurrency = throughput × latency. It's exact, assumption-free at steady state, and applies at every layer: in-flight requests, DB connections, queue depth, thread pools, GPU batch slots. Almost every capacity question reduces to applying it at the right layer.
- **Utilization and latency are enemies.** Queueing theory's one non-negotiable lesson: as utilization ρ → 1, wait time grows like 1/(1−ρ). At 50% busy, queues are negligible; at 80% they're noticeable; at 90–95% latency has exploded and small load ripples become outages. This hockey stick is why nothing latency-sensitive should be provisioned to run "efficiently" at 95%.
- **Provision for the peak, pay attention to the ratio.** Diurnal peak/trough of 2–5× is typical for consumer traffic; seasonal/event peaks (launch, Black Friday, viral moment) add another multiple. Capacity is bought for the p99 *day*; the peak-to-average ratio is what determines whether autoscaling/serverless (high ratio) or steady reserved capacity (low ratio) is economical.
- **Order of magnitude is the deliverable.** The goal of estimation is to distinguish 10 from 100 from 1000 — not 240 from 260. Round aggressively (1 day ≈ 10⁵ s, 1 month ≈ 2.6×10⁶ s ≈ 2.5M s), state assumptions inline, and sanity-check by computing the same quantity two independent ways.

## The numbers an architect carries (stable ballparks)

- **Latency hierarchy:** RAM reference ~100ns; SSD random read ~50–150µs; datacenter round-trip ~0.5ms; disk-backed DB query (indexed) ~1–10ms; same-continent WAN RTT ~20–80ms; cross-ocean ~100–200ms. Consequence: anything doing N sequential cross-service calls has a latency floor of N × RTT — chattiness, not CPU, is the usual latency budget-killer.
- **Throughput per core:** a well-written stateless HTTP service does ~1k–10k simple req/s per core (JSON in/out, no I/O wait); with a DB call per request, effective per-core throughput is set by *downstream latency and connection limits*, not CPU. Nginx/Envoy proxying: tens of thousands req/s/core. Interpreted-language app servers: hundreds to low-thousands req/s/core. Use 1k req/s/core as the pessimistic default for envelope math and refine by measurement.
- **Databases:** single beefy Postgres/MySQL primary handles ~5k–20k simple transactions/s and low-hundreds of GB working set comfortably; default `max_connections` is ~100–500 and each Postgres connection is a process — **connection count, not QPS, is the first limit apps hit**. Redis: ~100k+ ops/s single-threaded. Kafka: hundreds of MB/s per broker, ordered per partition.
- **Storage arithmetic:** 1M events/day × 1KB ≈ 1GB/day ≈ 365GB/yr raw — ×2–3 for indexes+replication+overhead. Logs/telemetry usually dwarf business data 10:1; estimate them separately or the storage estimate is fiction.
- **People-scale anchors:** 1M DAU with 20 requests/user/day ≈ 20M req/day ≈ 230 req/s average ≈ ~700–1200 req/s at diurnal peak. Memorize this shape: *DAU → daily requests → ÷86,400 → ×3–5 for peak*.

## Little's Law, worked

`L = λ × W`. Three directions, all useful:
1. **How many concurrent workers/connections?** API at 2,000 req/s, p50 latency 120ms → average concurrency = 2000 × 0.12 = **240 in-flight requests**. If each holds a DB connection for 40ms of that → 2000 × 0.04 = **80 concurrent DB connections** — over Postgres's comfortable default; you need a pooler (PgBouncer/RDS Proxy) *by arithmetic, before any incident*.
2. **What throughput can this fixed pool sustain?** 32 worker threads, each job takes 250ms → λ = L/W = 32/0.25 = **128 jobs/s max** — at *100% utilization*; apply the headroom rule and plan on ~90–100 jobs/s.
3. **How bad is the backlog?** Queue draining at 500 msg/s with 900k backlog → 30 minutes to drain *if* arrival stops; with arrivals at 400/s, net drain 100/s → 2.5 hours. Little's Law turns incident guesswork into an ETA.
Corollary: latency degradation *silently eats capacity* — if downstream latency doubles, the same worker pool sustains half the throughput. Systems fail sideways like this: a slow dependency turns into "we're out of threads" upstream.

## Peak-to-average and growth modeling

- Compute three numbers for any workload: **average rate, daily-peak rate (p99 hour), event-peak rate** (launch/marketing/viral). Provision baseline for daily peak + headroom; have a plan (autoscaling, pre-warming, load shedding, queueing) for event peak rather than owning it 24/7.
- **Linear growth** (sales-driven B2B): extrapolate with a safety factor; revisit quarterly. **Viral/compounding growth** (consumer): plan in doublings — the question is "what breaks at 2×, 4×, 8×?" and each answer usually differs (2×: DB connections; 4×: primary write IOPS; 8×: the single-region architecture). Write the breakpoints down; that list *is* the capacity plan.
- **Runway alerts:** for every hard limit (disk, connection cap, IP space, partition count, quota), alert on *projected time-to-exhaustion* (e.g., <90 days at trailing-30-day growth rate), not on percent-full. 80%-full disk growing 1%/year is fine; 40%-full growing 5%/week is an incident in six weeks.
- Cloud quotas are capacity limits too: on-demand instance quotas, Lambda concurrency, API rate limits, EIP counts. The p99-day plan that ignores a default quota gets to discover it during the event.

## Headroom policy — why 70–80%, not 95%

The utilization target debate resolves on four grounds:
1. **Queueing:** M/M/1 wait scales ~ρ/(1−ρ). Going 70%→90% utilization multiplies queue delay ~4×; 90%→95% doubles it again. Latency-SLO services should target **60–75%** at daily peak; batch/throughput systems can run **85–95%** because they optimize for utilization, not wait time.
2. **Failure absorption:** N+1/N+2 across zones means surviving instances absorb the dead zone's share. Three AZs at 66% each = 100% on two after one dies — i.e., *already saturated during the failure*. Target per-zone utilization ≤ (N−1)/N × latency-safe-target.
3. **Autoscaling lag:** scaling takes 1–5 min (VM boot, image pull, warmup); headroom is the buffer that pays for that lag. Faster spike-onset → more headroom or pre-scaling.
4. **Variance:** the 70% is 70% at *peak*, measured p95 — a system averaging 70% with bursty arrivals is intermittently at 100 (bursts hide inside minute-averaged metrics).
State the policy per tier: e.g., "stateless API: scale-out at 65% CPU; DB: alert at 60% because scaling it is a project, not an event."

## Load testing that predicts reality

- **Coordinated omission is the classic invalidator:** closed-loop tools that wait for each response before sending the next slow *down* during server stalls, silently dropping the very samples that show the stall — reported p99 can be off by orders of magnitude. Use **open-loop** (constant-arrival-rate) tools or corrected modes: wrk2, Vegeta, k6 (arrival-rate executors), Gatling (open injection profiles). If the tool's request rate droops when the server slows, its tail latencies are fiction.
- **Test the shapes that break systems, not just the plateau:** ramp (find the knee where latency departs linearity — that knee is your true capacity, not the point of first errors), **spike** (0→peak in seconds: tests autoscaling lag and cold paths), **soak** (hours at realistic load: finds leaks, log-disk fill, connection-pool decay, GC drift — soak failures are invisible in 10-minute runs), and **overload** (past capacity: does it degrade — shed load, queue, backpressure — or collapse?).
- Realism requirements: production-like *data volume* (a 100-row table lies about a 100M-row table's query plans), cache-realistic key distribution (Zipf, not uniform — uniform underestimates hot-key contention and overestimates cache hit rate), think-times and session mixes for user-facing flows, and TLS/network paths matching production.
- Deliverable of a load test is a sentence like: "knee at 3,200 req/s on 4×c7g.xlarge with p99 180ms; linear to that point; overload sheds gracefully to 2,800" — plus the graph. A test reporting only "handled 5k req/s" without tail latency at that rate reports nothing.

## Database sizing specifics

- Size by four axes independently — the binding constraint is usually not the one people discuss: **working set vs RAM** (if hot data + hot indexes fit in memory, reads are cheap; the cliff when they stop fitting is brutal — estimate working set, not total data), **write IOPS** (every write hits WAL + data + each index; replication multiplies it), **connections** (Little's Law from app concurrency; pooler mandatory beyond a few hundred), **storage growth** (with index+bloat multiplier ~2–3× raw).
- Read scaling: cache first (a 90% hit rate cuts DB read load 10×), then read replicas (mind replication lag for read-your-writes flows), then sharding — sharding is a last resort priced in engineer-years, and the envelope math frequently shows you're a decade from needing it.
- Quick envelope: 5M-DAU app, 20 reads + 2 writes per user-day → ~1,150 reads/s and ~115 writes/s average; ~400/s writes at peak. One well-tuned Postgres primary with a cache handles this with room — the correct design conclusion is "no exotic database required," which is the estimate's whole value.

## How an expert thinks through this

*"We're launching a notification service: 8M users, average 3 notifications/user/day, mobile push + in-app inbox, 'must handle a Super Bowl ad moment.' Size it."*

Baseline: 24M notifications/day ≈ 280/s average, ~1k/s diurnal peak. Tiny. The interesting number is the ad moment: marketing wants a blast to all 8M users "at once." Sending 8M pushes in, say, 10 minutes = ~13k sends/s — that's the real design load, 13× the daily peak. First insight: this is a *batch* disguised as a spike — nobody perceives a push arriving 4 minutes into the window, so the queue absorbs it. Reject "provision the fleet for 13k/s always" — 13× capacity for a monthly event is exactly what queues are for.

Workers: each send is an APNS/FCM call, ~50ms with batching. Little's Law: 13k/s × 0.05s = 650 concurrent sends → at 50 concurrent per worker (async I/O), ~13 workers; run 20 (headroom + zone loss). These scale from the normal-load 2–3 on queue depth — check autoscaling lag against the blast ramp; pre-scale before scheduled blasts instead of trusting reactive scaling (spike-onset seconds, scale-out minutes — the arithmetic says pre-scale).

Inbox writes: 8M inserts in 10 min ≈ 13k rows/s. *This* is the actual hard part — that's real write IOPS on the primary plus index maintenance. Options: batch inserts (COPY/multi-row, 10–50× cheaper per row), partition by day (also makes retention = partition drop, not DELETE), or fan-out-on-read for the blast case (store the blast once, materialize per-user on inbox open) — take batched inserts + daily partitions now, note fan-out-on-read as the 5× growth escape hatch. Storage: 24M/day × ~500B ≈ 12GB/day raw ≈ ×2.5 → ~11TB/yr — so retention policy is a launch requirement, not a later problem; 90-day retention caps it at ~2.7TB. Fine.

Downstream limits check: FCM/APNS rate limits and connection guidance at 13k/s — read the current quotas before promising the 10-minute window; if the platform caps us lower, the window stretches and *that's a product conversation, not an engineering one*. Sanity check by second path: 8M pushes ÷ 13 workers ÷ 50 concurrent ÷ (1/0.05s) ≈ 10.2 min ✓ consistent. Deliverables: the three load numbers, the breakpoint list ("2×: fine; 5×: inbox write path → fan-out-on-read; 10×: push-provider quotas"), and runway alerts on queue drain rate and partition disk.

## Failure modes & pitfalls

- **Sizing to the average.** 280/s average vs 13k/s event peak above — a 46× error. Always produce avg / daily-peak / event-peak; designs that quote one number are hiding the ratio.
- **Coordinated omission in the load-test report:** JMeter-style closed loop at fixed thread count showing p99=40ms while production shows 900ms stalls. Re-run open-loop at fixed arrival rate before believing any tail number.
- **Averages hiding saturation:** 60%-CPU minute-averages over 10-second 100% bursts; p99 latency already degraded. Look at max/p95 of fine-grained samples for anything with an SLO.
- **Connection math skipped:** 400 app instances × pool size 20 = 8,000 connections aimed at a Postgres set to 500. Little's Law on the DB-holding time first, pooler second, `max_connections` bump *last* (each connection costs server memory).
- **Load-testing with cold caches and toy data** → either wildly pessimistic (no cache warm) or wildly optimistic (everything fits in RAM). Production-scale data, Zipf keys, warmed steady state — then measure.
- **The 95%-utilization "efficiency" target on a latency service** — the hockey stick guarantees that normal variance produces timeouts. Efficiency targets belong on batch tiers; SLO tiers buy headroom.
- **Ignoring the failure-mode capacity:** N zones at 80% each = 120% on N−1. If you can't state utilization *during* a zone loss, the multi-AZ story is decorative.
- **Percent-full alerts instead of time-to-exhaustion** — the disk that jumps from 40% to full in a fortnight sails under every 80% threshold until it doesn't.
- **Soak-test allergy:** every "passed load test, died Saturday" story is a leak, fd exhaustion, log-volume fill, or token expiry that only hours-long runs at realistic load reveal. One soak per major release minimum.
- **Extrapolating linearly through a knee:** "we do 1k/s at 30% CPU, so 3k/s at 90%" — false the moment any queue, lock, or downstream approaches saturation; capacity is the measured knee, not a CPU proportion.
- **Forgetting retries in peak math:** at the worst moment, clients retry — a 2× retry storm rides on top of event peak. Budget it, and make load shedding + jittered backoff part of the overload plan.

## Worked micro-example — envelope for a URL-shortener-style read path

Given 50M redirects/day: ≈ 580/s avg → ~2k/s daily peak → assume 5k/s event peak. Latency budget 20ms. Redis lookup ~0.3ms → one modest Redis node (>50k ops/s) covers reads with 10× margin; concurrency at the edge = 5000 × 0.02s = 100 in-flight → 2–3 small app instances *for correctness*, deploy 4 across 2 zones for N+1 (per-zone loss check: 4→2 instances still ≥100 in-flight capacity? 2 × 64-conn ≈ fine). Storage: 50M/day writes? No — writes are link *creations*, maybe 1M/day × 200B ≈ 73GB/yr — trivially one Postgres. Total system: 4 small VMs, 1 cache, 1 small DB. The envelope's conclusion is architectural: **no distributed anything required** — and that conclusion took ten minutes.

## Verification / self-check

1. Every estimate shows its arithmetic and states assumptions inline; a reader can re-derive it.
2. Cross-checked by a second independent path (per-user math vs per-second math should agree within 2–3×).
3. All three load numbers present (avg / daily peak / event peak) plus the breakpoint list at 2×/4×/8×.
4. Little's Law applied at each pooled resource (threads, connections, workers) — no pool sized by folklore.
5. Headroom policy stated per tier with its reason (queueing, zone loss, scaling lag).
6. Any tail-latency claim traced to an open-loop measurement, not a closed-loop tool default.
Stopping rule: the envelope is done when the *decision* it feeds is insensitive to your remaining uncertainty (if the answer is "one Postgres" whether it's 300/s or 900/s, stop refining); precision beyond decision-sensitivity is procrastination — go measure the real system instead.
