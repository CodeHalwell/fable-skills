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
- **Coordination is the expensive thing.** Every design sits on a spectrum from "no coordination, converge eventually" (CRDTs, gossip) to "coordinate every write" (consensus, 2PC). Cost grows with coordination scope and frequency. The expert move is shrinking the thing that needs coordination (e.g., only leader election goes through Raft; the data plane doesn't), not choosing a stronger protocol everywhere.
- **State is the enemy; idempotence is the weapon.** Stateless components restart freely and scale trivially; every piece of distributed state needs a replication story, a failover story, and a consistency story. Push state into as few, as boring components as possible (one database, one consensus service) and make everything around them idempotent and restartable.

## Decision frameworks

### CAP/PACELC, applied not recited

CAP only bites during a partition, and the real everyday tradeoff is PACELC's "else": latency vs consistency when the network is fine.

| Situation | Choose | Because |
|---|---|---|
| Read-heavy, staleness tolerable (product pages, timelines) | AP-style replicas, async replication | Users can't perceive 500ms staleness; they perceive 500ms latency |
| Uniqueness/invariants (usernames, account balance ≥ 0, inventory) | Single-writer per key or consensus-backed CP store | Invariants can't be repaired by convergence; duplicates/negatives are business incidents |
| Cross-region writes, invariant needed | Partition the keyspace so each key has one home region | Global synchronous consensus costs a WAN round-trip per write (~100–300ms); per-key home region pays it only for remote keys |
| "We need strong consistency everywhere" | Push back; enumerate the 2–3 operations that actually need it | Blanket linearizability is a latency tax on every operation for the sake of a few |

### Do you actually need consensus (Raft/Paxos/etcd/ZooKeeper)?

- Legitimate uses: leader election, distributed locks with correctness stakes, cluster membership/config, a linearizable register. That's roughly the whole list.
- You do NOT need it for: work distribution (use a queue), dedupe (use a DB unique constraint), or "shared config" that tolerates seconds of staleness (use a DB + polling).
- A distributed lock alone is insufficient for mutual exclusion on an external resource: the old holder may be paused (GC, VM migration) and act *after* its lease expired. Pair the lock with a **fencing token** — a monotonically increasing epoch number issued with the lock and checked by the resource, which rejects writes carrying a smaller token than the largest it has seen — or accept the race explicitly.
- If you find yourself implementing consensus by hand ("we'll have the nodes vote"), stop: use etcd/ZooKeeper or a database row with compare-and-set. Homemade quorum protocols are where split-brain bugs are born.

### Saga vs 2PC for multi-service writes

Default to sagas. 2PC blocks all participants on a coordinator that can crash between prepare and commit, holds locks across the network for the full protocol duration, and most infrastructure (HTTP services, most queues) can't participate anyway.

Saga design rules:
- A saga is a sequence of local transactions, each with a **compensating action** (refund, release, cancel) that semantically undoes it.
- Order steps so the hardest-to-compensate step goes **last**: charge the card after reserving inventory, not before — refunds are painful, releasing a reservation is free.
- Every compensation must itself be idempotent and retried until success; a compensation that can silently fail is a money-losing design.
- Intermediate states are visible to other readers (a "PENDING" order). Model them explicitly as states in the domain rather than pretending atomicity; consumers must handle them.
- Persist saga progress (an orchestrator table or event stream) so a crashed coordinator resumes rather than abandoning half-done work.

### Dual-write problem → outbox pattern

Never write to your DB and then publish to Kafka/SNS as two separate operations — the process can die between them, and no ordering of the two is safe (DB-then-publish loses events; publish-then-DB emits events for rolled-back writes).

- Insert the event into an `outbox` table **in the same local transaction** as the state change.
- A relay (poller, or CDC via Debezium reading the WAL) publishes from the outbox and marks rows sent.
- Guarantees: at-least-once publication, correct order per aggregate. Consumers still dedupe on event ID — the relay can crash between publish and mark-sent.
- The same reasoning applies to any two-system write (DB + search index, DB + cache, DB + email): one of them must be driven asynchronously *from* the other, not written alongside it.

### Retries

- Retry only idempotent operations, or operations made idempotent via an idempotency key the server dedupes on.
- Use exponential backoff **with full jitter** — `sleep(random(0, min(cap, base * 2**attempt)))` — because deterministic backoff synchronizes clients into waves.
- Cap total attempts and honor a retry budget (e.g., retries ≤ 10% of request volume); when the budget is exhausted, fail fast instead of queueing more retries.
- Retry at ONE layer of the stack (usually the edge or the service mesh — not both, and not every intermediate service), or a 3-deep call chain with 3 retries each amplifies one failure into 27 requests.
- Distinguish retryable errors (timeout, 503, connection refused) from non-retryable (400, 401, business rejection); retrying a validation error just burns budget.

### Backpressure

- Every unbounded queue is an outage postponed. Bound every queue; when full, choose deliberately:
  - **Block the producer** — propagates backpressure upstream; usually right for internal pipelines.
  - **Shed load** — right at the edge, with 429 + `Retry-After`.
  - **Drop oldest** — right for metrics/telemetry where fresh beats complete.
- "The queue absorbs bursts" is true only if drain rate exceeds average arrival rate; otherwise the queue just adds latency to an eventual collapse.
- Watch queue *wait time*, not just depth: a short queue draining slowly is saturated; a deep queue draining fast is fine.

## Failure modes & pitfalls

- **Retry storms / metastable failure.** A brief blip → timeouts → every layer retries 3× → downstream sees 27× load → stays saturated forever even after the original cause is gone. Corrections: retries at one layer only, full jitter, retry budgets, circuit breakers that trip on error *rate* not consecutive errors. When debugging "the incident outlasted the trigger", suspect this first.
- **Timeout-then-retry on non-idempotent POST.** Classic double-charge. Correction: idempotency key generated by the *original* caller, stored server-side with the response, replayed on duplicate key. The key must survive the caller's own crash-and-retry (persist it before first send), and the server must store key → response atomically with the side effect.
- **Split-brain after leader lease expiry.** Old leader is GC-paused for 40s, lease expires, new leader elected, old leader wakes and writes. Corrections: fencing tokens validated by the storage layer; or old leader re-checks its lease *with margin* before each write (still racy without fencing — say so). If a design has "leader" and no fencing story, flag it.
- **Two primaries because the failure detector was too aggressive.** Short health-check timeouts trade fewer stalls for more false positives, and a false-positive failover is how you get dual primaries accepting divergent writes. Failover decisions need quorum agreement behind them, not a single node's opinion of another's health.
- **Ordering assumed across partitions/keys.** Kafka orders within a partition only; SQS standard queues don't order at all; two events for the same entity routed by different keys will interleave. Correction: partition by entity ID, and make consumers tolerate reordering across entities.
- **Consumer rebalance duplicates.** Kafka consumer processes a batch, crashes before committing offsets → the batch is redelivered to another consumer. Committing offsets *before* processing flips it to data loss instead. There is no setting that avoids both — idempotent processing is the only real fix.
- **Reading your own write from a stale replica.** User updates profile, GET goes to a lagging replica, "my change disappeared". Corrections: session stickiness to the writer for N seconds, read-your-writes tokens (client sends last-seen LSN/version; replica waits or proxy routes), or just read the primary for the user's own recent data.
- **Dedupe window shorter than max retry horizon.** Idempotency keys kept 24h, but a stuck consumer replays a 3-day-old backlog → duplicates. Dedupe retention must exceed the maximum possible redelivery delay (queue retention + DLQ replay), or dedupe must be a permanent unique constraint.
- **Compensations that can fail without a plan.** Saga step 3 fails; compensation for step 2 also fails (refund API down) → money stuck. Compensations must be durably enqueued and retried forever, with an alert + manual-intervention queue as the true fallback. If the saga design says "then we roll back" without persistence, it loses money.
- **Health checks that lie.** A node that answers `/healthz` (process up) but can't reach its DB stays in the LB pool serving 500s. Health checks should probe the dependency the traffic needs — but NOT so deeply that one dependency outage marks every node unhealthy and the LB removes the whole fleet. Use min-healthy thresholds / fail-open on total health-check failure.
- **Thundering herd on recovery.** Dependency comes back; every client's breaker half-opens at once, every queued retry fires, and the freshly recovered service is immediately re-killed. Corrections: jittered breaker probes, gradual ramp (token-bucket admission on recovery), and draining backlogs at a controlled rate.
- **Clock-based token/cache expiry across machines.** Issuer says token valid until T; validator's clock is 30s behind; token honored 30s too long (or rejected while valid). Build skew allowance into validation windows and never enforce sub-minute precision across machines.
- **Fan-out without a concurrency bound.** One request fans out to 50 backends in parallel; tail latency of the slowest dominates (at 50 calls, hitting a per-call p99 is a coin flip — 39% chance at least one call exceeds it), and a slow backend consumes all upstream threads. Corrections: bound concurrency, hedge only cheap idempotent reads, set the fan-out deadline from the caller's remaining budget and pass deadlines downstream so doomed work is cancelled.

## Worked micro-examples

### 1. Idempotent consumer for at-least-once delivery (Postgres)

```sql
-- Consumer processing payment events; event_id is the dedupe key.
BEGIN;
INSERT INTO processed_events (event_id) VALUES ($1)
  ON CONFLICT (event_id) DO NOTHING;
-- rowcount 0 => duplicate: COMMIT and ack without side effects.
UPDATE accounts SET balance = balance - $2 WHERE id = $3;
COMMIT;  -- ack the message only after commit
```

The dedupe insert and the side effect share one transaction, so a crash between them is impossible. Ack-after-commit means a crash before ack yields redelivery, which the dedupe row absorbs. This is "exactly-once processing" — and it works *because* the side effect lives in the same database as the dedupe table. If the side effect is an external API call, you can't get this; you need that API to accept an idempotency key instead.

### 2. Full-jitter backoff (the version that prevents storms)

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

`random.uniform(0, backoff)` (full jitter) — not `backoff + small_jitter` — because after a mass failure, thousands of clients computed the same schedule; full jitter spreads the retry wave uniformly across the window instead of shifting the wave later.

### 3. Why wall clocks can't order events — concrete failure

Node A (clock +80ms fast) writes `x=1` at true time 12:00:00.000, stamping 12:00:00.080. Node B writes `x=2` at true time 12:00:00.050, stamping 12:00:00.050. Last-write-wins by timestamp keeps `x=1` — the *earlier* write wins and B's later write is silently discarded. Nothing crashed, no partition, replication healthy. Fix: version numbers per key (compare-and-set on version), or route all writes for a key through one node.

### 4. Fencing token mechanics

Lock service issues `(lock, token=33)` to client A; A pauses 40s (GC); lease expires; service issues `(lock, token=34)` to B; B writes to storage with token 34; storage records 34. A wakes, writes with token 33; storage compares 33 < 34 and **rejects**. Without the storage-side check, A's stale write lands — this is why the *resource*, not the lock service, must validate the token, and why locks over resources that can't check tokens (a plain HTTP API, a file share) are advisory at best.

## Verification / self-check

Before presenting a distributed design or diagnosis, walk it through this gauntlet:

- **Kill test:** for each arrow in the diagram, kill the process on either end mid-operation. What state results? Who cleans it up, and when?
- **Duplicate test:** deliver every message twice, out of order. Same end state?
- **Pause test:** freeze any node for 60s (GC/VM migration), then resume. Does it act on stale beliefs (expired lease, old leadership, cached routing)?
- **Storm test:** a dependency returns 100% errors for 2 minutes. Does load on it go up (retry amplification) or down (breaker/budget)? Does the system recover *by itself* when the dependency returns, without being re-crushed by the backlog?
- **Clock test:** skew every clock ±200ms. Does anything mis-order, expire early, or double-fire?
- **Claims audit:** if you claimed "exactly-once", point to the dedupe key, its storage, and its retention window. If you claimed a transaction across services, point to each compensation and its retry mechanism. If you claimed mutual exclusion, point to the fencing check at the resource.
