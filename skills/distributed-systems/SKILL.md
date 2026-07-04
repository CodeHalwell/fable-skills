---
name: distributed-systems
description: Load when designing, reviewing, or debugging systems that span multiple processes or machines — microservices, message queues, replication, consensus, distributed transactions, retries, or any bug involving timeouts, duplicates, reordering, or split-brain. Also load when someone asks for "exactly-once delivery" or a cross-service transaction.
---

# Distributed Systems Reasoning

## Core mental model

- **Partial failure is the defining property.** On one machine, a crash kills everything at once; distributed, some parts keep running while others are dead, slow, or unreachable — and you cannot distinguish "crashed", "slow", and "network dropped my packet" from the outside. Every design question reduces to: *what does each node do when it can't tell what happened elsewhere?* If a design doc doesn't answer this, it isn't a design.
- **A timeout is a guess, not a fact.** When an RPC times out, the operation may have executed. Any retry is therefore a potential duplicate, and any "abort" is potentially aborting something that succeeded. Design every write path assuming the caller will see a timeout and retry.
- **Exactly-once delivery is a lie; exactly-once *processing* is achievable.** Networks give you at-most-once or at-least-once. Pick at-least-once and make the receiver idempotent (dedupe key, conditional write, or naturally idempotent operation). Anyone selling "exactly-once" is doing at-least-once + dedupe under the hood — find the dedupe key and its retention window; that window is where the guarantee silently expires.
- **Wall clocks cannot order events.** NTP skew of tens of milliseconds is normal; VM pauses and leap-second smearing make it worse. Two timestamps from different machines closer than ~100ms tell you nothing about causal order. Use per-key monotonic versions, Lamport/vector clocks, or a single sequencer (e.g., one Kafka partition) when order matters. `last_write_wins` on wall-clock timestamps = silent data loss under skew.
- **Coordination is the expensive thing.** Every design sits on a spectrum from "no coordination, converge eventually" (CRDTs, gossip) to "coordinate every write" (consensus, 2PC). Cost grows with coordination scope and frequency. The expert move is shrinking the thing that needs coordination (e.g., only leader election goes through Raft; data plane doesn't), not choosing a stronger protocol everywhere.

## Decision frameworks

**CAP/PACELC, applied not recited.** CAP only bites during a partition, and the real everyday tradeoff is PACELC's "else": latency vs consistency when the network is fine.

| Situation | Choose | Because |
|---|---|---|
| Read-heavy, staleness tolerable (product pages, timelines) | AP-style replicas, async replication | Users can't perceive 500ms staleness; they perceive 500ms latency |
| Uniqueness/invariants (usernames, account balance ≥ 0, inventory) | Single-writer per key or consensus-backed CP store | Invariants can't be repaired by convergence; duplicates/negatives are business incidents |
| Cross-region writes, invariant needed | Partition the keyspace so each key has one home region | Global synchronous consensus costs a WAN round-trip per write (~100–300ms); per-key home region costs it only for remote keys |
| "We need strong consistency everywhere" | Push back; enumerate the 2–3 operations that actually need it | Blanket linearizability is a latency tax on every operation for the sake of a few |

**Do you actually need consensus (Raft/Paxos/etcd/ZooKeeper)?** Only for: leader election, distributed locks with correctness stakes, cluster membership/config, or a linearizable register. You do NOT need it for: work distribution (use a queue), dedupe (use a DB unique constraint), or "shared config" that tolerates seconds of staleness (use a DB + polling). And a distributed lock alone is insufficient for mutual exclusion on an external resource — the old holder may be paused (GC, VM migration) and act after its lease expired. Pair the lock with a **fencing token** (monotonic epoch number checked by the resource) or accept the race.

**Saga vs 2PC for multi-service writes.** Default to sagas. 2PC blocks all participants on a coordinator that can crash between prepare and commit, holds locks across the network, and most infrastructure (HTTP services, most queues) can't participate anyway. Use a saga: sequence of local transactions, each with a **compensating action** (refund, release, cancel). Design rules: order steps so the hardest-to-compensate step goes **last** (charge the card after reserving inventory, not before); every compensation must itself be idempotent and retried until success; accept that intermediate states are visible (a "PENDING" order) and model them explicitly rather than pretending atomicity.

**Dual-write problem → outbox pattern.** Never write to your DB and then publish to Kafka/SNS as two separate operations — the process can die between them, and no ordering of the two is safe (DB-then-publish loses events; publish-then-DB emits events for rolled-back writes). Instead: insert the event into an `outbox` table **in the same local transaction** as the state change; a relay (poller or CDC/Debezium) publishes from the outbox and marks rows sent. This gives at-least-once publication with correct ordering per aggregate; consumers dedupe on event ID.

**Retries.** Retry only idempotent operations (or operations made idempotent via an idempotency key the server dedupes on). Use exponential backoff **with full jitter** — `sleep(random(0, min(cap, base * 2**attempt)))` — because deterministic backoff synchronizes clients into waves. Cap total attempts and honor a retry budget (e.g., retries ≤ 10% of request volume); when the budget is exhausted, fail fast. Never retry on a connection you know delivered the request unless you can dedupe.

**Backpressure.** Every unbounded queue is an outage postponed. Bound every queue; when full, choose deliberately: block the producer (propagates backpressure upstream — usually right for internal pipelines), shed load (right at the edge, with 429 + Retry-After), or drop oldest (right for metrics/telemetry). "The queue absorbs bursts" is true only if drain rate exceeds average arrival rate; otherwise the queue is just adding latency to an eventual collapse.

## Failure modes & pitfalls

- **Retry storms / metastable failure.** A brief blip → timeouts → every layer retries 3× → downstream sees 27× load → stays saturated forever even after the original cause is gone. Corrections: retries at ONE layer only (usually the edge or a service mesh, not both), full jitter, retry budgets, circuit breakers that trip on error *rate* not consecutive errors. When debugging "the incident outlasted the trigger", suspect this first.
- **Timeout-then-retry on non-idempotent POST.** Classic double-charge. Correction: idempotency key generated by the *original* caller, stored server-side with the response, replayed on duplicate key. The key must survive the caller's own crash-and-retry (persist it before first send).
- **Split-brain after leader lease expiry.** Old leader is GC-paused for 40s, lease expires, new leader elected, old leader wakes and writes. Corrections: fencing tokens validated by the storage layer; or old leader self-checks lease *with margin* before each write (still racy without fencing — say so). If a design has "leader" and no fencing story, flag it.
- **Two nodes both think they're primary because the failure detector used a too-short timeout.** Aggressive timeouts trade fewer stalls for more false positives, and false-positive failover is how you get dual primaries. Failover decisions need consensus/quorum behind them, not a single node's opinion of another's health.
- **Ordering assumed across partitions/keys.** Kafka orders within a partition only; SQS standard queues don't order at all; two events for the same entity routed by different keys will interleave. Correction: partition by entity ID, and make consumers tolerate reordering across entities.
- **Reading your own write from a stale replica.** User updates profile, GET goes to a lagging replica, "my change disappeared". Corrections: session stickiness to the writer for N seconds, read-your-writes tokens (client sends last-seen LSN/version, replica waits or proxy routes), or just read the primary for the user's own data.
- **Dedupe window shorter than max retry horizon.** Idempotency keys kept 24h, but a stuck consumer replays a 3-day-old backlog → duplicates. The dedupe retention must exceed the maximum possible redelivery delay (queue retention + DLQ replay), or dedupe must be a permanent unique constraint.
- **Compensations that can fail without a plan.** Saga step 3 fails, compensation for step 2 also fails (refund API down) → money stuck. Compensations must be durably enqueued and retried forever, with an alert + manual-intervention queue as the true fallback. If the saga design says "then we roll back" without persistence, it loses money.
- **Health checks that lie.** A node that answers `/healthz` (process up) but can't reach its DB stays in the LB pool serving 500s. Health checks should probe the dependency the traffic needs — but NOT so deeply that one dependency outage marks every node unhealthy and the LB removes the whole fleet (min-healthy thresholds / fail-open).
- **Clock-based token/cache expiry across machines.** Issuer says token valid until T; validator's clock is 30s behind; token honored 30s too long (or rejected while valid). Build skew allowance into validation windows and never enforce sub-minute precision across machines.

## Worked micro-examples

**1. Idempotent consumer for at-least-once delivery (Postgres):**

```sql
-- Consumer processing payment events; event_id is the dedupe key.
BEGIN;
INSERT INTO processed_events (event_id) VALUES ($1)
  ON CONFLICT (event_id) DO NOTHING;
-- rowcount 0 => duplicate: COMMIT and ack without side effects.
UPDATE accounts SET balance = balance - $2 WHERE id = $3;
COMMIT;  -- ack the message only after commit
```

The dedupe insert and the side effect share one transaction, so a crash between them is impossible. Ack-after-commit means a crash before ack yields redelivery, which the dedupe row absorbs. This is "exactly-once processing" — and it works because the side effect lives in the same database as the dedupe table. If the side effect is an external API call, you can't get this; you need the API to accept an idempotency key instead.

**2. Full-jitter backoff (the version that prevents storms):**

```python
import random, time

def call_with_retry(fn, base=0.1, cap=20.0, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError:
            if attempt == max_attempts - 1:
                raise
            time.sleep(random.uniform(0, min(cap, base * 2 ** attempt)))
```

`random.uniform(0, backoff)` (full jitter) — not `backoff + small_jitter` — because after a mass failure, thousands of clients computed the same schedule; full jitter spreads the retry wave uniformly across the window instead of shifting it.

**3. Why wall clocks can't order events — concrete failure.** Node A (clock +80ms fast) writes `x=1` at true time 12:00:00.000, stamping 12:00:00.080. Node B writes `x=2` at true time 12:00:00.050, stamping 12:00:00.050. Last-write-wins by timestamp keeps `x=1` — the *earlier* write wins and B's later write is silently discarded. Nothing crashed, no partition, replication healthy. Fix: version numbers per key (compare-and-set on version), or route all writes for a key through one node.

## Verification / self-check

Before presenting a distributed design or diagnosis, walk it through this gauntlet:
- **Kill test:** for each arrow in the diagram, kill the process on either end mid-operation. What state results? Who cleans it up?
- **Duplicate test:** deliver every message twice, out of order. Same end state?
- **Pause test:** freeze any node for 60s (GC/VM migration), then resume. Does it act on stale beliefs (expired lease, old leadership)?
- **Storm test:** dependency returns 100% errors for 2 minutes. Does load on it go up (retry amplification) or down (breaker/budget)? Does the system recover *by itself* when the dependency returns?
- **Clock test:** skew every clock ±200ms. Does anything mis-order, expire early, or double-fire?
- If you claimed "exactly-once", point to the dedupe key, its storage, and its retention window. If you claimed a transaction across services, point to each compensation and its retry mechanism.
