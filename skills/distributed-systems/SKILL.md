---
name: distributed-systems
description: Load when designing, reviewing, or debugging systems that span multiple processes or machines — microservices, message queues, replication, consensus, distributed transactions, retries, or any bug involving timeouts, duplicates, reordering, or split-brain. Also load when someone asks for "exactly-once delivery" or a cross-service transaction.
---

# Distributed Systems Reasoning

This is a correction sheet, not a survey. Standard patterns (outbox, sagas, fencing tokens, full-jitter backoff, idempotency keys) are assumed known; they appear only as compressed anchors. The expanded sections are where the standard answers are incomplete or subtly wrong.

## Compressed anchors (apply without re-derivation)

- Timeout = unknown, not failure. Every write path assumes the caller retries → idempotency key persisted by the caller *before* first send, stored server-side atomically with the side effect, response replayed on duplicate.
- "Exactly-once delivery" = at-least-once + dedupe. Find the dedupe key and its retention window; the window is where the guarantee silently expires. Retention must exceed the max redelivery horizon — queue retention + DLQ replay + backfill/reconciliation jobs — or be a permanent unique constraint.
- Any two-system write (DB + Kafka, DB + cache, DB + search, DB + email): outbox row in the same local transaction, relay/CDC publishes. Consumers still dedupe on event ID — the relay can crash between publish and mark-sent.
- Multi-service writes: saga, not 2PC. Hardest-to-compensate step last (charge after reserve). Compensations idempotent, durably enqueued, retried until success, manual-intervention queue as the true fallback. PENDING states are real domain states consumers must handle.
- Distributed lock over an external resource is advisory unless the *resource* checks a monotonic fencing token.
- Retries: `sleep(random(0, min(cap, base * 2**attempt)))` (full jitter), at ONE layer of the stack, budget ≤ ~10% of request volume, retryable errors only.
- LWW on wall-clock timestamps = silent data loss under NTP skew; order with per-key versions, one sequencer per key, or HLC.
- Kafka rebalance: commit-after-processing duplicates, commit-before loses; no config avoids both — idempotent processing (or offset committed in the same DB transaction as the effect) is the only fix.
- Consensus (etcd/ZK/Raft) is for leader election, locks with correctness stakes, membership, a linearizable register — not for work distribution (queue), dedupe (unique constraint), or staleness-tolerant config (DB + polling).

## Corrections to the standard answers (the delta)

- **Deep health checks cause fleet-wide self-removal.** The reflex fix for "node answered /healthz but couldn't reach its DB" is "make health checks probe dependencies." Done naively, one dependency outage marks *every* node unhealthy and the LB removes the whole fleet — converting a partial degradation into a total outage. Corrections: probe the dependency the traffic actually needs, but configure min-healthy thresholds / fail-open when health-check failures exceed a fleet fraction (most LBs support this; it's off by default), and keep liveness (restart me) separate from readiness (route to me).
- **Failover decisions need quorum, not one observer.** A single node's opinion that the primary is dead is how you get two primaries: short health-check timeouts trade fewer stalls for more false positives, and a false-positive failover under network jitter creates divergent writes. The failure detector's *decision* must go through quorum agreement (or the consensus service), even if the probes themselves are cheap.
- **Circuit breakers: trip on error *rate* over a window, never on N consecutive errors.** Consecutive-error breakers almost never trip under interleaved traffic (one success resets the count during a 40% error storm) and flap under sparse traffic. Use rate-over-window with a minimum request count.
- **Hedged requests are not a general tail-latency fix.** The textbook advice ("send a backup request at p95, take the first response") is safe only for cheap, idempotent reads. Hedging writes duplicates side effects; hedging expensive calls adds load precisely when the system is slow — the hedge becomes the storm. Before hedging: bound the fan-out concurrency, set the fan-out deadline from the caller's *remaining* budget, and propagate deadlines downstream so doomed work is cancelled. (Fan-out arithmetic: 50 parallel calls each with p99 500ms → ~39% of requests exceed 500ms; the fix ladder is reduce fan-out, bound concurrency, then hedge reads only.)
- **Recovery is a designed event, not the absence of failure.** When a dependency returns after a blip, every client's breaker half-opens at once and every queued retry fires — the freshly recovered service is immediately re-killed (metastable failure: the outage outlasts its trigger). Corrections: jittered breaker probe timing, token-bucket admission ramp on recovery, drain backlogs at a controlled rate, and drop already-timed-out work from queues (deadline-aware or LIFO/CoDel-style) instead of burning capacity on requests nobody is waiting for. Test the recovery: inject a 2-minute dependency blip and verify the system returns to steady state *unassisted*.
- **Clock skew allowance is a design parameter, not a hope.** Issuer-validator pairs on different machines must never enforce sub-minute expiry precision (tokens, cache entries, leases). Build explicit skew windows into validation; a validator 30s behind honors a token 30s too long, and "we run NTP" does not bound VM-pause or leap-smear excursions.
- **Read-your-own-writes needs a mechanism, not a hope:** session stickiness to the writer for N seconds, a last-seen-LSN token the replica waits on, or primary reads for the user's own recent data. "Replication is usually fast" is the bug report waiting to be filed.

## Verification gauntlet (run before presenting any design or diagnosis)

- **Kill test:** for each arrow in the diagram, kill either end mid-operation. What state results, who cleans it up, when?
- **Duplicate test:** deliver every message twice, out of order. Same end state?
- **Pause test:** freeze any node 60s (GC/VM migration), resume. Does it act on stale beliefs (expired lease, old leadership, cached routing)?
- **Storm test:** a dependency returns 100% errors for 2 minutes. Does its load go up (retry amplification) or down (breaker/budget)? Does the system recover *by itself* without being re-crushed by backlog?
- **Clock test:** skew every clock ±200ms. Anything mis-ordered, early-expired, double-fired?
- **Claims audit:** "exactly-once" → point to the dedupe key, storage, retention window. Cross-service transaction → point to each compensation and its retry mechanism. Mutual exclusion → point to the fencing check *at the resource*.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 9 baseline (cut/compressed to anchors), 3 partial (sharpened), 0 delta.
- Baseline already produces cold: outbox/CDC, orchestrated sagas with correct step ordering, fencing tokens with storage-side checks, full-jitter + one-retry-layer + 10% budgets, idempotency-key retention vs DLQ-replay horizon, LWW clock-skew loss, Kafka offset dilemma, fan-out tail math and metastable-failure mechanisms.
- Biggest gaps found: deep-health-check fleet self-removal (fail-open/min-healthy), breaker trip criterion (rate vs consecutive), hedging safety caveats (idempotent-reads-only, concurrency bounds), recovery-ramp design.
