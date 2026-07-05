---
name: serverless-architectures
description: Load when designing, reviewing, or debugging serverless systems — Lambda/Azure Functions/Cloud Run functions, event-driven pipelines with queues and event buses, Step Functions/Durable Functions orchestration, cold start problems, idempotency, DLQs, serverless cost or lock-in questions, or deciding whether serverless fits a workload at all.
---

# Serverless Architecture Judgment

## Core mental model

- **Serverless is a cost/ops shape, not a technology.** You buy scale-to-zero, per-invocation billing, and zero capacity management; you pay with cold starts, duration caps, per-unit premiums at sustained load, and a distributed system whether you wanted one or not.
- **Everything is at-least-once; idempotency is the correctness baseline**, not an optimization.
- **Functions compose through durable intermediaries** (queues, topics, buses, streams, state machines), never by synchronous function→function chains — that's the distributed monolith.
- **Lock-in lives in the events and IAM, not the compute.** Judge a design's lock-in by counting event-source types and permission edges, not function count. Deep coupling to Step Functions/DynamoDB streams bought consciously for velocity is fine — record it as a decision with an exit cost.

## Fit decision

| Workload signal | Verdict |
|---|---|
| Spiky API, ms–s handlers, idle nights | FaaS |
| Queue/stream consumers, batch glue | FaaS (cold starts irrelevant there — don't optimize a non-problem) |
| Steady ≥50% duty-cycle traffic | Containers (per-invocation premium compounds) |
| WebSockets/long-lived connections | Containers or managed WS layer |
| Heavy deps, >15 min, >10GB, GPU | Serverless containers / batch |
| Sub-50ms p99 always-hot API | Containers (or paid-for warm FaaS) |

**The middle ground most "serverless API" workloads actually belong in (2026):** containers-on-demand (Cloud Run request-concurrency model, ACA/KEDA, Fargate) — scale-to-zero with container ergonomics, concurrency >1 per instance to amortize expensive init (loaded ML models), no FaaS caps, no toolchain fork for container-shipping teams. The reflex "serverless = Lambda" undersells this tier.

## State management without servers

Name where every piece of state lives; "in the function" is never an answer. Workflow state → orchestrator; aggregation/join state → conditional writes on one item (never read-modify-write); session → external store with TTL; rate/dedup/locks → conditional writes with TTL (never in-memory token buckets); large payloads → claim check from day one (256KB queue/state-machine limits surface mid-incident, not in review). Draw the pipeline; for each arrow and box write the state's home and TTL — unhomed state is where serverless designs corrupt data.

## Cold start engineering

Ladder, cheapest first: (1) runtime/package trimming, lazy imports, clients initialized outside the handler; (2) **SnapStart** (Java 11+, Python 3.12+, .NET 8+ as of 2026 — free for Java, cache+restore fees for Python/.NET; requires published versions; incompatible with provisioned concurrency and EFS; re-seed RNG and reconnect in restore hooks); (3) provisioned concurrency / always-ready instances sized to the steady *floor*, scheduled to diurnal load; (4) accept them for async paths — and *state that explicitly* in the design so nobody "fixes" it later. Warmer crons are obsolete; their presence in a design signals stale patterns — audit the rest of it accordingly.

## Event-driven composition rules

- DLQ on every async consumer, alarm on depth, redrive path built before the first incident; stream sources get `bisectBatchOnFunctionError`, `MaximumRetryAttempts`, `MaximumRecordAgeInSeconds`, on-failure destination, and `ReportBatchItemFailures` — together they turn "shard stalled 24h behind one poison record" into "record DLQ'd, shard advances."
- Idempotency key from the *business event*, never the message ID (redeliveries share it; republishes don't); persist key→result via conditional write with TTL **longer than the longest possible redelivery path — including human-driven DLQ redrives the next morning**, which is the case that breaks the textbook "hours" answer. Lambda Powertools `@idempotent` implements this; don't hand-roll.
- Ordering: FIFO `MessageGroupId` or per-entity stream partitioning caps throughput per group — design for commutativity (versioned last-write-wins) when you can.
- Count end-to-end retry amplification for every side effect: SNS→3 queues→3 retries = 12+ executions of one event against your non-idempotent payment API.
- HTTP front door: Function URLs < HTTP APIs < REST APIs (only for validation/usage plans/private endpoints); everything caps ~29–30s — long work returns `202` + status resource.
- **Deployment safety:** functions deploy in milliseconds, which tempts teams to skip progressive delivery. Use versions + aliases with weighted canary (CodeDeploy `Canary10Percent5Minutes`-style) and auto-rollback on alarm — the alias flip is the fastest rollback mechanism in computing; wire it.

## Orchestration vs choreography

Events between bounded contexts, orchestration within one. Orchestrate when a flow has >2–3 steps *and* an owner/deadline (visible state, per-step retries, compensation); choreograph broadcast-shaped reactions. Standard workflows bill per transition (a 10k-iteration loop is ~$0.50/run and risks the 25k-event history limit — use Express or distributed Map); Express ≤5 min, at-least-once. Durable Functions orchestrators replay — determinism violations (direct I/O, `DateTime.Now`, `Guid.NewGuid()`) are the #1 bug; all effects in activities, time/GUIDs from the context APIs.

## Observability

Correlation ID propagated through every hop (API request ID → message attribute → function context) or debugging is archaeology across 12 log groups. Trace by default, sampled. Alert on *system* signals — DLQ depth, queue oldest-age, iterator age, error rate per alias, concurrency-vs-limit — not per-function noise. Log ingestion is the serverless tax: retention set at creation, verbose paths sampled.

## The distributed-monolith failure mode

Tells: function A sync-invokes B invokes C; deploys require coordinating three functions; five functions write one table; an event schema change breaks four consumers at runtime. Cause: decomposing by technical layer instead of business capability. Corrections: merge layer-functions into one handler (a function is a *deployment* unit, not a code-organization unit — three steps in one function is fine); version event schemas with consumers tolerant of additive change; one writer per table/stream. Ask of any function: "can it deploy alone, and does it own its data?"

## Local dev and testing strategy

Thin handler + pure core. Unit-test the core with zero cloud; test handler wiring with *recorded real event payloads* (envelope shapes are gnarlier than docs — SQS `body` is a string of JSON, double-wrapped through SNS unless `RawMessageDelivery` is enabled). Emulators (LocalStack, `sam local`) are for iteration speed only — green-on-emulator verifies nothing about IAM, limits, or retry semantics, which are precisely the dominant serverless bug classes. The real integration tier is a per-developer/PR cloud stack (serverless idles at ~$0): publish real event → assert resulting state.

## How an expert thinks through this

*"Image SaaS: uploads → 5 renditions + AI tags → notify. Rails+Sidekiq today; 100k uploads/day, 10× influencer spikes."*

Shape check passes (bursty, event-driven, seconds of CPU per unit). Split by **failure domain, not step count**: renditions (pure CPU) and tagging (slow, rate-limited model endpoint) each get their own EventBridge target, SQS buffer, DLQ, retry policy — coupling them makes rendition latency inherit tagging's failures. Rejected Step Functions: steps are independent, no compensation — a state machine would add per-transition cost and a false sequential dependency. But "notify when all parts done" is genuinely stateful → DynamoDB counter with conditional update; a state machine for a single join is machinery without payoff (revisit if compensation or deadlines appear). Idempotency key = `object_etag + rendition_size` (users re-upload the same file; S3 events are at-least-once). Cold starts: all async — explicitly ignored. CPU-bound rendition function: more memory is often *cheaper* (CPU scales with memory; same GB-s, lower wall time — never leave CPU-bound functions at low memory). Notify calls legacy Rails: retries + circuit breaker + DLQ so its downtime never backs up the pipeline.

## Failure modes & pitfalls (checklist)

- No DLQ + FIFO/stream source + one poison message = stalled shard and a silent SLA breach.
- Timeout inversion: function timeout > gateway's 29s → 504s, work continues, clients retry, work duplicates.
- Lambda-in-VPC → RDS without RDS Proxy: connection exhaustion at first burst.
- Account/regional concurrency is shared blast radius: reserve for critical functions, cap bulk consumers (doubles as DB-protection throttle).
- **Recursive invocation:** S3 event → function writes to the same bucket/prefix → runaway bill overnight. Separate input/output prefixes and scope event filters. AWS recursive-loop detection halts ~16-deep chains through SQS/SNS **and, since late 2024, S3** — pre-2025 knowledge says S3 loops aren't detected; they now are — but detection covers only some topologies (not EventBridge or DynamoDB streams), so design it out rather than rely on it.
- Event source mapping batch × per-record p99 must fit the timeout with margin, or perpetual partial-batch failures.
- EventBridge targets retry up to 24h/185 attempts then **silently drop** — configure a DLQ per target and alarm it like a queue DLQ.
- Idempotency TTL shorter than the redrive horizon: next-morning human redrive re-executes side effects.
- Cost surprise at success: $50/mo FaaS scaling linearly to $15k/mo at 300× while containers would flatten — re-run the crossover arithmetic at every order of magnitude.

## Worked micro-example — retry policy as orchestration, not code (Step Functions ASL)

```json
"ChargeCard": {
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": { "FunctionName": "charge-card", "Payload.$": "$" },
  "TimeoutSeconds": 15,
  "Retry": [{
    "ErrorEquals": ["Lambda.TooManyRequestsException", "States.Timeout"],
    "IntervalSeconds": 2, "MaxAttempts": 4, "BackoffRate": 2.0, "JitterStrategy": "FULL"
  }],
  "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "RefundAndFail", "ResultPath": "$.error" }],
  "Next": "FulfillOrder"
}
```
The judgment encoded: retries with backoff+jitter live in the state machine (visible, tunable, no redeploy), not in handler code; the timeout is explicit and shorter than the caller's; the unhappy path is a named, testable compensation state rather than an exception handler someone hopes fires.

## Verification / self-check

1. Every async edge: DLQ, redrive path, alarm.
2. Every handler answers "this exact event arrives twice" with a mechanism, not a hope.
3. Timeouts strictly decrease downstream; retry amplification counted per side effect.
4. Payloads bounded by claim-check.
5. Cost computed at 1× and 10×; FaaS-vs-container crossover located.
6. One test tier runs against real cloud resources with recorded event shapes.
Stopping rule: when each edge's *failure* path is designed and a poison message demonstrably ends in an alarmed DLQ, stop hardening — further resilience patterns are speculative complexity.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails SnapStart runtimes/cost/caveats, the anti-warmer position, Powertools idempotency, Kinesis poison-record settings, Standard/Express economics incl. the 25k history limit, raw message delivery, EventBridge silent drops, memory-CPU scaling, Durable Functions determinism, and lock-in layering — all compressed to anchors.
- The one correction kept sharp: Opus asserts S3-triggered recursive loops are *not* covered by AWS loop detection; S3 was added in late 2024 (coverage still partial — design loops out regardless). Idempotency-TTL-vs-human-redrive and the "state your ignored cold starts explicitly" discipline were the only other above-baseline sharpenings.
