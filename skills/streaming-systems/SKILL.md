---
name: streaming-systems
description: Load when designing or debugging stream processing — Kafka topics/partitions/consumer groups, Flink or Kafka Streams jobs, event-time windowing, delivery semantics, schema evolution on streams, consumer lag, or when deciding whether a "realtime" requirement actually needs streaming at all.
---

# Stream Processing Engineering

Frontier models already hold the Kafka/Flink fundamentals cold — partition-key consequences, EOS scope, watermark idleness, KIP-848/KIP-890, ForSt, sink idempotency. This sheet is the compressed checklist plus the judgment calls and phrasings that still separate designs that survive from ones that don't.

## Anchors (one line each — expand only if challenged)

1. Kafka = partitioned immutable log: replay free, fan-out free, ordering *only within a partition*.
2. **The partition key sets three things at once** — per-key ordering, the parallelism ceiling (max useful consumers = partitions), and hot-spot risk. Most streaming design mistakes are partition-key mistakes.
3. EOS (idempotent producers + transactions; hardened by KIP-890 in Kafka 4.0) covers **Kafka→processor→Kafka only**; every exit to a DB/API/email is at-least-once — design every sink for duplicates.
4. Event-time windows + watermarks derived from *measured* p99 lateness; idle inputs stall the min-watermark (`withIdleness(...)`) — when a windowed job silently stops emitting, check watermark progress before the sink.
5. Streaming state is a database you now operate: every keyed state needs TTL or windowed retention; checkpoint duration << interval; Flink 2.0's ForSt disaggregated backend decouples recovery time from state size.
6. Consumer lag trend is the single health metric; backpressure propagating to lag is *designed* behavior — the failure is an unbounded in-process queue converting it into an OOM.

## The batch-would-do test (say it early; it saves a year of ops)

Freshness requirement in *minutes from the data's consumer* (not the stakeholder's adjective)? → micro-batch every 5 min is a batch job. Genuine streaming = something *acts* within seconds (fraud blocks, pricing, alerting), per-event triggers, or a shared event backbone. Fourth question that gets skipped: can this team operate a stateful 24/7 job (checkpoints, rescaling, replay runbooks)? If not, the streaming version will be *less* reliable than the batch it replaced.

## Delivery semantics — the decision order

1. What breaks on duplicates? Keyed upserts don't → at-least-once + idempotent sink is the default answer, simpler and faster than transactions.
2. What breaks on loss? `acks=all` + idempotent producer; commit offsets *after* the side effect.
3. Kafka-to-Kafka atomic read-process-write → EOS (`exactly_once_v2` / Flink exactly-once sink). State the costs out loud: latency tied to commit/checkpoint interval, and downstream **must set `isolation.level=read_committed`** (the client default is `read_uncommitted` — aborted records leak silently).
4. Kafka-to-external → dedupe-at-sink: stable event ID end-to-end, `INSERT ... ON CONFLICT DO NOTHING`/MERGE. 2PC into external systems: rejected — fragile, slow, and you needed the dedupe key anyway for replays.
5. **The check reviewers skip:** is the dedupe key unique per *business event* or per *attempt*? Sink idempotency is only as good as the key's semantics — dedupe on the business key.

## Kafka 4.0 / Flink 2.0 anchors (verified 2026)

- KIP-890: transaction protocol strengthened (per-transaction epoch bump; zombie-fencing edge cases closed).
- KIP-848: broker-driven incremental rebalancing, GA — `group.protocol=consumer`; only moving partitions pause. Still: keep groups small/stable, and `max.poll.interval.ms` ejection → rebalance storm remains — slow processing goes in a separate thread pool with manual offsets or smaller polls.
- Flink 2.0: ForSt disaggregated state on S3/HDFS for very large state; incremental checkpoints and TTL first.

## Design-review checklist (fail any → not done)

- Partition key audited against all three jobs (ordering / ceiling / measured skew). Hot key → composite key (`merchant_id + order_id % N`), *explicitly accepting* loss of per-merchant total order in the design doc.
- Never increase partition count on a keyed topic — key→partition mapping breaks mid-history (ordering + Streams state locality). Headroom up front; scaling properly = new topic + replay.
- Watermark delay (postpones first firing) vs allowed lateness (keeps state alive to re-fire) distinguished; beyond-lateness events side-outputted to a late topic and **counted** — the drop rate audits your lateness budget.
- Window choice: tumbling default; hopping multiplies state/output by size/hop; session = data-driven boundaries, most state-hungry. "Latest value per key" wants no window at all — it's a table (compacted topic / KTable; stream-table duality).
- `enable.auto.commit=true` with process-after-poll = silent loss; commit manually after processing, let the idempotent sink absorb crash-window duplicates.
- Replay discipline: business logic uses the event's embedded timestamp so replays regenerate identical results; runbook (`kafka-consumer-groups.sh --reset-offsets --to-datetime` or fresh group) *executed once* before needed.
- Retention decided per topic (default 7 days = bugs older than a week unrecoverable); raw copy in the lake so batch can always rebuild.
- Schema registry rejects breaking changes at CI time; compatibility mode by *who upgrades first* (BACKWARD = consumers first; FORWARD = producers first); genuinely breaking change = new topic version + dual-write — you cannot atomically upgrade all consumers of a log that retains history.
- Alerts: lag *trend* and watermark staleness, not instantaneous spikes; stream-vs-sink daily reconciliation count for money-adjacent data.

## Debugging anchor: "double-counted downstream"

Query both sides first: Streams-side aggregate correct, Postgres high → duplication is at the sink boundary (consistent with the prior: EOS stops at Kafka's edge). Cause: connector restarts rewound to last committed offset and replayed. Fix at the sink (unique index + `ON CONFLICT DO NOTHING`), repair history, add the reconciliation monitor. Rejected: enabling Kafka transactions (don't reach Postgres; latency for zero fix), committing more often (smaller duplicates are still wrong numbers). Then the key-semantics check above.

## Worked micro-example: honest event-time windowing (Flink)

```java
WatermarkStrategy<Order> wm = WatermarkStrategy
    .<Order>forBoundedOutOfOrderness(Duration.ofMinutes(2))   // measured p99 lateness + margin
    .withIdleness(Duration.ofMinutes(1))                      // kills the window-that-never-closes
    .withTimestampAssigner((o, ts) -> o.eventTimeMillis);

stream.assignTimestampsAndWatermarks(wm)
      .keyBy(o -> o.merchantId)
      .window(TumblingEventTimeWindows.of(Time.minutes(5)))
      .allowedLateness(Time.minutes(10))
      .sideOutputLateData(lateTag)                            // counted, not vanished
      .aggregate(new OrderSum());
```

Every number is a *stated budget* someone can challenge with data — that, not the API calls, is the difference between engineering and defaults.

## Verification / self-check

- Trace one event: delivered twice? 3 hours late? consumer crashes mid-batch? redeploy? Each answer specific ("upserted on event_id") — "should be fine" means you don't know.
- State TTL'd; checkpoints well inside interval; replay rehearsed; lag/watermark/late-drop monitored; reconciliation exists.
- Stopping rules: freshness turned out to be minutes → ship the batch version and pocket the simplicity. Exactly-once beyond the Kafka boundary is not a goal; idempotent sinks are.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 11 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails: partition-key triple, EOS scope + KIP-890, KIP-848 + `group.protocol=consumer`, ForSt, watermark idleness fix, dedupe-at-sink, keyed-topic partition-increase hazard, compatibility-mode selection, auto-commit/read_uncommitted traps.
- Restructured to a correction sheet; retained value = the business-key-vs-attempt dedupe check, stated-budget framing, the can-the-team-operate-it question, and the review checklist as one place.
