---
name: streaming-systems
description: Load when designing or debugging stream processing — Kafka topics/partitions/consumer groups, Flink or Kafka Streams jobs, event-time windowing, delivery semantics, schema evolution on streams, consumer lag, or when deciding whether a "realtime" requirement actually needs streaming at all.
---

# Stream Processing Engineering

## Core mental model

1. **Kafka is a partitioned immutable log, not a queue.** Consumers own offsets and read at their own pace; the broker deletes nothing on read. Everything else follows: replay is free (rewind the offset), fan-out is free (another consumer group), ordering exists *only within a partition*.
2. **The partition key is the highest-leverage design decision.** It simultaneously determines ordering (per-key order guaranteed, cross-key order nonexistent), parallelism ceiling (max useful consumers = partition count), and hot-spot risk (skewed keys → one starving partition while others idle). You are choosing all three at once; most streaming design mistakes are really partition-key mistakes.
3. **At-least-once is reality; "exactly-once" is idempotency plus transactions wearing a suit.** Any system with retries delivers duplicates somewhere. Kafka's EOS (idempotent producers + transactions, strengthened in Kafka 4.0 via KIP-890, as of 2026) gives exactly-once *within the Kafka→processor→Kafka loop*; the moment data exits to an external database, API, or email, you're back to at-least-once and need an idempotent sink. Design every sink as if duplicates will arrive, because they will.
4. **Event time ≠ processing time, and the gap is unbounded.** Mobile clients sync hours late; a replayed backfill delivers last week "now". Correct aggregations require event-time windows plus watermarks — an explicit, chosen tolerance for lateness — not "whatever arrived in the last 5 minutes".
5. **Streaming state is a database you now operate.** Windows, joins, and dedupe all keep keyed state (Flink/Kafka Streams: RocksDB-class local stores, checkpointed to durable storage; Flink 2.0 adds ForSt disaggregated state on S3/HDFS for very large state, as of 2026). State size, checkpoint duration, and recovery time are your new capacity-planning axes.
6. **Consumer lag is the single health metric.** Lag stable near zero: healthy. Lag growing: the consumer is slower than the producer and *will* fall over — everything else (CPU, memory) is a symptom detail.

## When streaming is over-engineering: the batch-would-do test

Ask, in order: (1) What is the *actual* freshness requirement, in minutes, from the consumer of the data — not from the stakeholder's adjective "realtime"? Most "realtime" dashboards are 5-minute requirements; a micro-batch every 5 minutes is a batch job. (2) Does anything *act* on the data within seconds (fraud blocking, pricing, alerting)? That's the genuine streaming case. (3) Is the computation per-event or windowed-aggregate? Per-event triggers favor streaming; aggregates tolerate batch. (4) Can the team operate a stateful streaming job 24/7 — checkpoints, rescaling, replay runbooks? If not, the streaming version will be *less* reliable than the batch it replaced. Prior: when someone says "we need Kafka + Flink" for an hourly report, the correct design is a batch pipeline, and saying so saves a year of ops pain. Streaming earns its keep at seconds-level latency, per-event actions, or as a shared event backbone feeding many consumers.

## Delivery semantics honesty

Reasoning chain for "what semantics do I need?":
1. **What breaks on duplicates?** Counters and sums inflate; upserts keyed on a natural ID don't. If every sink write is a keyed upsert or has a unique constraint, at-least-once + idempotent sink is sufficient and *simpler and faster* than transactions. This is the default answer.
2. **What breaks on loss?** If nothing may be lost, producer must use `acks=all` and `enable.idempotence=true` (default in modern clients), and consumers must commit offsets *after* the side effect, never before. Committing before processing is the classic silent-loss config.
3. **Kafka-to-Kafka pipelines needing atomic read-process-write:** use EOS (`processing.guarantee=exactly_once_v2` in Kafka Streams; Flink's Kafka sink in exactly-once mode with checkpointing). Costs: transactional latency tied to commit interval, and downstream consumers must set `isolation.level=read_committed` — forgetting this silently re-exposes aborted records.
4. **Kafka-to-external-system:** transactions don't reach the external system. Pattern: **dedupe-at-sink** — carry a stable event ID end-to-end; sink does `INSERT ... ON CONFLICT (event_id) DO NOTHING` or a MERGE. This turns at-least-once into effective exactly-once and is more robust than any distributed-transaction cleverness. The two-phase-commit route into external systems is almost always rejected: fragile, slow, and the dedupe key was needed anyway for replays.

## Event time, watermarks, windows

- **Watermark = the stream's clock**: "I believe no events older than T remain." Set it from measured lateness (p99 event-time skew) plus margin, not vibes. Too tight → late events dropped/mis-windowed; too loose → results delayed and state balloons.
- **The window-that-never-closes bug:** if the watermark derives from event timestamps per partition and one partition goes quiet (low-traffic key, dead producer), the *minimum* across partitions stalls, no watermark advances, no window ever fires, and the job "runs fine" while emitting nothing. Fix: idleness handling (Flink `withIdleness(...)` on the watermark strategy) or bounded-out-of-orderness watermarks with idle-source timeout. When a windowed job silently stops producing output, check watermark progress *first*, before suspecting the sink.
- **Allowed lateness vs watermark delay:** watermark delay postpones the *first* firing; allowed lateness keeps window state around to re-fire updates for stragglers. Beyond allowed lateness, events land in a side output — route it to a dead-letter/late-events topic and *count it*; the drop rate tells you if your lateness budget is honest.
- **Window selection reasoning:** tumbling for non-overlapping reporting buckets ("orders per 5 min"); hopping/sliding when you need smoothness or SLA-style "over any 10-minute span" semantics (note: each event lands in size/hop windows — state and output multiply accordingly); session windows for activity bursts with a gap timeout (user sessions, device incidents) — the only window type whose boundaries are data-driven, hence the most state-hungry and merge-prone. If the consumer just wants "latest value per key", you don't want windows at all — you want a compacted topic / table view (stream-table duality: a table is a stream's latest-state-per-key; a stream is a table's changelog; Kafka Streams KTable and compacted topics are this made concrete).

## State, checkpointing, backpressure

- Budget state size *before* building: keys × per-key bytes × windows retained. A dedupe-forever store over an unbounded key space is a slow-motion OOM — every keyed state needs TTL or windowed retention. State-size explosion is the most common way streaming jobs die at month three.
- Checkpoint duration must be << checkpoint interval. When checkpoints start timing out, recovery time and end-to-end latency (under EOS) degrade together; that's the signal to enable incremental checkpoints, cut state (TTL), or (Flink 2.0, as of 2026) move to disaggregated ForSt state where recovery time decouples from state size.
- **Backpressure is end-to-end or it's an outage.** A slow sink propagates: sink stalls → operator buffers fill → source stops polling → consumer lag grows. That's the *designed* behavior — lag is the shock absorber. The failure mode is an unbounded in-process queue "fixing" backpressure by converting it into an OOM. Diagnose backpressure at the *last* backpressured operator (Flink UI shows this); the bottleneck is the first non-backpressured thing downstream of it — usually the sink or a skewed key.
- **Rebalancing tax:** every consumer join/leave/deploy triggers partition reassignment; with the legacy protocol this stops the world. Kafka 4.0's KIP-848 broker-driven protocol (GA, enable with `group.protocol=consumer`, as of 2026) makes rebalances incremental — only moving partitions pause. Still: keep consumer groups small-and-stable, beware `max.poll.interval.ms` — a consumer processing one batch too slowly gets ejected, triggering a rebalance, which slows others, which ejects more: the rebalance storm. Slow processing belongs in a separate thread pool with manual offset management, or in smaller poll batches.

## Schema evolution on streams

A topic is an API with many unknown consumers; producers can't coordinate deploys with all of them. Use a schema registry (Confluent SR or compatible; Avro/Protobuf/JSON-Schema) with a compatibility mode chosen *by who upgrades first*: BACKWARD (new reader reads old data — consumers upgrade first; the common default), FORWARD (old reader reads new data — producers upgrade first), FULL (both, most constrained). The registry rejects breaking registrations at *produce/CI time* — that's the point: it converts a 3am consumer crash into a failed build. Rules of thumb: add optional fields with defaults freely; never rename or retype a field (add new + deprecate old); a genuinely breaking change means a *new topic version* (`orders.v2`) with dual-write or a migrator, because you cannot atomically upgrade all consumers of a log that retains history.

## How an expert thinks through it: "payment events are being double-counted downstream"

Internal monologue: *Double-counted where — in the Kafka Streams aggregate, or in the Postgres sink table?* Query both: the Streams-side count is correct; Postgres is high. *So the duplication is at the sink boundary — consistent with the prior: EOS covers Kafka-to-Kafka, sinks are at-least-once.* Check the sink connector: it does plain INSERTs and the connector task restarted twice yesterday (deploys) — each restart rewound to the last committed offset and replayed a few hundred events. *Considered: turn on Kafka transactions.* Rejected — transactions don't extend into Postgres; they'd add latency and change nothing here. *Considered: commit offsets more often to shrink the replay window.* Rejected as a mitigation-not-fix — smaller duplicates are still wrong numbers. Correct fix: the events carry `payment_id`; make the sink idempotent — unique index on `payment_id`, `INSERT ... ON CONFLICT DO NOTHING` — then *repair* the table by deduping historical rows, and add a monitor comparing stream-side vs sink-side daily counts. One more check before closing: is `payment_id` actually unique per business event, or per attempt? (If per attempt, dedupe on the business key instead — sink idempotency is only as good as the key's semantics.)

## Failure modes & pitfalls

- **Keying by high-skew field** (e.g., `merchant_id` where one merchant is 40% of traffic): one hot partition caps throughput regardless of consumer count. Detect: per-partition lag/throughput spread. Fix: composite key (`merchant_id + order_id % N`) *accepting* you lose per-merchant total ordering — say so explicitly, don't discover it later.
- **Increasing partition count on a keyed topic** silently breaks key→partition mapping: same key now lands elsewhere, per-key ordering and Streams state locality break across the boundary. Plan partition counts with headroom up front; scaling a keyed topic properly means a new topic + replay.
- **Auto-commit loss:** `enable.auto.commit=true` with processing after poll can commit offsets for records that then fail in-process — silent loss. Commit manually after successful processing (accepting duplicates on crash, handled by the idempotent sink).
- **`isolation.level=read_uncommitted`** (the default) downstream of a transactional producer: consumers see aborted transactional records. Anything reading EOS output must set `read_committed`.
- **Replay without event-time discipline:** reprocessing last week through logic that stamps `processing_time` corrupts aggregates. All business logic uses the event's embedded timestamp; replays then regenerate identical results — replay is your backfill mechanism, treat it as first-class (documented runbook: new consumer group or offset reset via `kafka-consumer-groups.sh --reset-offsets --to-datetime`, idempotent sink absorbs the rewrite).
- **Retention as an afterthought:** default 7-day retention means a bug older than a week is unrecoverable from the stream. Decide per-topic: long retention or compaction for source-of-truth topics; and land a raw copy in the lake (cheap) so batch can always rebuild.
- **Treating lag alerts as noise:** page on lag *trend* (growing for >N minutes) and on watermark staleness, not on instantaneous lag spikes from deploys.

## Worked micro-example: honest event-time windowing (Flink, Python-esque pseudocode of the real API)

```java
WatermarkStrategy<Order> wm = WatermarkStrategy
    .<Order>forBoundedOutOfOrderness(Duration.ofMinutes(2))   // measured p99 lateness + margin
    .withIdleness(Duration.ofMinutes(1))                      // prevents the window-that-never-closes
    .withTimestampAssigner((o, ts) -> o.eventTimeMillis);     // event time, never ingestion time

stream.assignTimestampsAndWatermarks(wm)
      .keyBy(o -> o.merchantId)
      .window(TumblingEventTimeWindows.of(Time.minutes(5)))
      .allowedLateness(Time.minutes(10))                      // stragglers re-fire updates
      .sideOutputLateData(lateTag)                            // beyond that: counted, not vanished
      .aggregate(new OrderSum());
```

Every number above is a *stated budget* (2 min skew, 10 min lateness) that someone can challenge with data — that's the difference between engineering and defaults.

## Verification / self-check

- Trace one event end-to-end and answer: what happens if it's delivered twice? Arrives 3 hours late? The consumer crashes mid-batch? The job is redeployed? Each answer must be specific ("upserted on event_id", "side-output to late-topic, counted") — "should be fine" means you don't know.
- Confirm the partition key against all three of its jobs: ordering needed? parallelism sufficient? skew measured?
- State has TTL or windowed retention, checkpoints complete well inside their interval, and there's a written replay runbook that has been executed at least once.
- Lag, watermark progress, and late-drop rate are monitored; a stream-vs-sink reconciliation count exists for money-adjacent data.
- Stopping rule: if the freshness requirement turned out to be minutes, stop — ship the batch/micro-batch version and pocket the operational simplicity. Within a genuine streaming system, stop hardening when duplicates, lateness, and replay all have boring rehearsed answers; exactly-once beyond the Kafka boundary is not a goal, idempotent sinks are.
