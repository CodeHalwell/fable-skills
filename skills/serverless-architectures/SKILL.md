---
name: serverless-architectures
description: Load when designing, reviewing, or debugging serverless systems — Lambda/Azure Functions/Cloud Run functions, event-driven pipelines with queues and event buses, Step Functions/Durable Functions orchestration, cold start problems, idempotency, DLQs, serverless cost or lock-in questions, or deciding whether serverless fits a workload at all.
---

# Serverless Architecture Judgment

## Core mental model

- **Serverless is a cost/ops shape, not a technology.** You're buying: scale-to-zero, per-invocation billing, and no capacity management — paying with: cold starts, execution time caps, per-unit premiums at sustained load, and a distributed system whether you wanted one or not. Every "should this be serverless?" question reduces to whether the workload's shape matches that trade.
- **The shape that fits:** spiky or unpredictable traffic, event-driven triggers, short stateless units of work, low duty cycle, teams that want ops surface near zero. **The shape that doesn't:** steady high throughput (the per-invocation premium compounds — at constant load FaaS costs multiples of containers), long-lived connections (WebSockets held in-process, DB connection pools), latency floors tighter than cold-start mitigation can hit, and monolithic transactions spanning many steps.
- **Everything is at-least-once.** Queue triggers, event buses, stream shards, retries — every serverless event source can and will deliver duplicates. Idempotency is not an optimization; it's the correctness baseline. Design every handler assuming it runs twice with the same event.
- **Functions don't compose by calling each other.** Synchronous function→function chains inherit every downstream cold start and failure, multiply cost (caller pays for waiting), and rebuild the monolith with network calls inside — the *distributed monolith*. Functions compose through **durable intermediaries**: queues, topics, event buses, streams, state machines.
- **Lock-in lives in the events and IAM, not the compute.** Porting a handler's code between clouds is an afternoon; porting the EventBridge rules, IAM/RBAC graph, SQS semantics, DynamoDB streams, and Step Functions definitions is the actual migration. Judge lock-in by counting event sources and permission edges, not lines of function code.

## Fit decision — the questions in order

1. **Duty cycle:** what fraction of the hour is real work happening? <20–30% → serverless economics win. >50% sustained → containers with commitments are cheaper; run the numbers, don't assume either way.
2. **Unit-of-work duration:** seconds → fine. Minutes → still fine (check the cap: Lambda 15 min). Hours, or needs GPU / >10GB memory → not FaaS; use serverless *containers* (Fargate tasks, ACA Jobs, Cloud Run jobs) or batch services.
3. **Latency sensitivity of the first byte:** user-facing p99 SLO under ~200ms with idle periods → cold starts are a real engineering problem (below). Queue/stream consumers → cold starts are irrelevant; anyone optimizing them there is polishing a non-problem.
4. **Connection model:** long-held stateful connections (WebSockets, gRPC streams, DB pools per instance) → FaaS fights you; either use the platform's managed WebSocket layer (API Gateway WebSocket APIs, Azure Web PubSub, SignalR) or use containers.
5. **State between steps:** none → functions. Workflow state across steps → orchestration layer (Step Functions / Durable Functions), never in-memory or "in the queue message that grows forever."

**FaaS vs serverless containers:** prefer containers-on-demand (Fargate/ACA/Cloud Run) over FaaS when: dependencies are heavy (multi-GB images, native libs), work units exceed FaaS caps, you need concurrency >1 per instance to amortize expensive initialization (loaded ML model serving many requests), or the team already ships containers and FaaS would fork the toolchain. Cloud Run's request-concurrency-per-instance model and ACA's KEDA scaling give scale-to-zero with container ergonomics — as of 2026 this middle ground is where most "serverless API" workloads actually belong.

## Cold start engineering (state of 2026)

Attack in this order — cheapest first:
1. **Runtime and package choice:** interpreted-but-light runtimes (Node, Python) cold-start in low hundreds of ms; JVM/.NET without help take seconds; Go/Rust are consistently fast. Trim deployment size (cold start scales with code+deps loading), lazy-import heavy libs, initialize clients outside the handler (reused across warm invocations).
2. **Lambda SnapStart** (as of 2026: Java 11+, Python 3.12+, .NET 8+): restores a snapshot of the *initialized* runtime, cutting multi-second inits to sub-second. Free for Java; Python/.NET incur cache + per-restore charges. Caveats: incompatible with provisioned concurrency and EFS; snapshot resume breaks naive uniqueness — RNG seeds, cached credentials, and connections initialized before the snapshot are duplicated across restores; use runtime hooks to re-seed/reconnect.
3. **Provisioned concurrency / always-ready instances** (Lambda provisioned concurrency; Azure Flex Consumption always-ready): pre-warmed instances billed while idle — you're buying back the latency by giving up scale-to-zero on that slice. Size to the *steady floor* of concurrency, let bursts overflow to on-demand cold starts. Schedule it (Application Auto Scaling) to follow diurnal load.
4. **Accept them:** for async paths, batch, internal tools — do nothing. State this explicitly in designs so nobody "fixes" it later.
Anti-pattern: cron "warmer" pings — they keep one instance warm while real traffic fans out to N cold ones; superseded by the mechanisms above.

## Event-driven composition rules

- **Between any two functions, put a durable thing:** SQS/Service Bus queue (point-to-point, competing consumers), SNS/Event Grid topic (fan-out), EventBridge/Event Grid bus (routing by content, third-party events), Kinesis/Event Hubs (ordered streams, replay). Direct sync invocation is reserved for request/response edges (API Gateway → function) — not for pipelines.
- **DLQs are mandatory on every async consumer.** No exceptions. A queue trigger without a DLQ silently discards poison messages after retries (or worse, retries forever, blocking FIFO/stream shards). Configure `maxReceiveCount` deliberately (3–5 typical), alarm on DLQ depth > 0, and build the redrive path *before* the first incident. For stream sources (Kinesis/DynamoDB streams), configure `on-failure` destinations and `bisectBatchOnFunctionError` — one poison record otherwise stalls the whole shard, which is the classic 3am serverless page.
- **Idempotency, concretely:** derive a stable idempotency key from the *business event* (order ID + state transition), not the message ID (redeliveries share message IDs, but *republishes* don't). Persist key → result with a conditional write (DynamoDB `attribute_not_exists`, or a unique constraint) and a TTL longer than your maximum redelivery horizon. AWS Lambda Powertools has a ready-made `@idempotent` decorator implementing exactly this — use it instead of hand-rolling.
- **Ordering:** most sources don't guarantee it. If you need per-entity order, you need FIFO queues with `MessageGroupId` = entity ID, or a stream partitioned by entity — and your throughput is now capped per group/shard. Design for commutativity instead when you can (version numbers, last-write-wins on versioned state).

**The synchronous front door:** for HTTP entry, prefer the lightest gateway that meets requirements — on AWS: Lambda Function URLs (single function, no fan-out features) < API Gateway HTTP APIs (cheaper, most REST use cases) < REST APIs (only for request validation, usage plans/API keys, private endpoints); ALB→Lambda when the fleet is mixed containers+functions. Every option caps integration timeout around 29–30s — long work must return `202 Accepted` + a status resource or WebSocket/polling; designs that stream a 3-minute job through an HTTP gateway are broken on arrival.

**Deployment safety:** functions deploy in milliseconds, which tempts teams to skip progressive delivery — don't. Use versions + aliases with weighted canary shifting (Lambda + CodeDeploy `Canary10Percent5Minutes`-style configs, or slot/revision splitting on Azure/Cloud Run), automatic rollback on alarm. The unit of rollback is the alias flip, and it's the fastest rollback mechanism in all of computing — wire it up.

## Orchestration vs choreography

- **Choreography** (services react to each other's events): loose coupling, great for *broadcast-shaped* flows — "order placed" → independent reactions (email, analytics, inventory). Fails when there's a *process* with an outcome someone owns: nobody can answer "where is order 123 stuck?", timeouts and compensation are smeared across services.
- **Orchestration** (Step Functions / Durable Functions): a named owner for a multi-step process with visible state, retries with backoff per step, timeouts, human-approval waits, saga compensation. Use it whenever a flow has >2–3 steps *and* an outcome with a deadline or an owner.
- Rule of thumb: **events between bounded contexts, orchestration within one.** A state machine spanning five teams' services recreates central coupling; events for a single tightly-ordered process recreate the debugging nightmare.
- Step Functions specifics: Standard workflows bill per state transition and support year-long executions with exactly-once semantics; Express workflows bill per duration/memory, run ≤5 min, at-least-once — high-volume short pipelines belong on Express, long/durable business processes on Standard. Don't put a 10k-iteration loop in a Standard workflow (per-transition pricing bites); use a Map state in distributed mode or Express child workflows.
- Durable Functions' orchestrator constraint: orchestrator code **replays** — it must be deterministic (no `DateTime.Now`, no direct I/O, no random) — all effects go through activities. Violating this yields nondeterminism errors and corrupted orchestrations; it is the #1 Durable Functions bug.

## Observability — non-negotiables for systems made of 40 small pieces

- **Structured JSON logs with a correlation ID propagated through every hop** (API request ID → queue message attribute → function context). Without it, debugging a pipeline is archaeology across 12 log groups. Lambda Powertools Logger / Azure Functions + App Insights do the propagation if you let them.
- **Distributed tracing on by default** (X-Ray/OTel, App Insights), sampled — the trace is the only artifact that shows *where* a 6-second user request spent its time across five functions and two queues.
- Alert on the *system* signals, not per-function noise: DLQ depth, queue age (oldest message), stream iterator age, error-rate per alias, and concurrency-vs-limit. Per-invocation error alarms on 40 functions produce fatigue, not insight.
- Log-ingestion cost is the serverless tax: hundreds of chatty functions at per-GB ingestion pricing. Set retention at creation, sample INFO+ in high-volume paths.

## The distributed-monolith failure mode

Symptoms: a "microservices" diagram where function A synchronously invokes B invokes C; every deploy requires coordinating three functions; one shared database table written by five functions; a change to one event schema breaks four consumers at runtime. Causes: decomposing by technical layer (validate-fn → transform-fn → save-fn) instead of by business capability, and using sync calls where events belong. Corrections: merge layer-functions into one handler (a function is a deployment unit, not a code-organization unit — three steps in one function is *fine*); schema-version events (registry or explicit `version` field, consumers tolerant of additive change); one writer per table/stream. Ask of any function: "can this deploy alone, and does it own its data?" If either answer is no for many functions at once, you've built a monolith with network partitions inside it.

## Local dev and testing strategy

Split the code so the strategy is possible: **thin handler** (parse event, call logic, shape response) + **pure core** (business logic, unit-testable with zero cloud). Then:
- Unit-test the core exhaustively — no emulators involved.
- Test handler wiring with *recorded real event payloads* (grab them from logs; SQS/EventBridge/API Gateway shapes are gnarlier than docs suggest) as fixtures.
- Emulators (LocalStack, Azurite, `sam local`) are useful for fast iteration on glue, but **do not treat green-on-emulator as verified** — IAM, service limits, and exact retry semantics differ. The layer emulators cover is precisely the layer they simulate imperfectly.
- The real integration test is a **dev stack in the cloud per developer/PR** (cheap by construction — serverless idles at ~$0), exercised end-to-end: publish real event → assert on resulting state. Budget CI time for this; it's the only test tier that catches IAM and event-shape bugs, which are the dominant serverless bug class.

## How an expert thinks through this

*"Image-processing SaaS: users upload photos, we generate 5 renditions + AI tags, then notify. Currently a Rails monolith with Sidekiq; 100k uploads/day, 10× spikes when an influencer posts."*

Shape check: bursty ✓, event-driven ✓ (upload event), short units ✓ (each rendition seconds of CPU) — serverless-shaped. Duty cycle: 100k × ~30s of total work/day ≈ 35 CPU-hours/day spread unevenly — low duty cycle with violent spikes; autoscaling containers would either lag the spike or idle expensively. FaaS it is.

Composition: upload → S3 event → *one function doing all 5 renditions + tags?* No — the AI tagging calls a model endpoint (slow, rate-limited) while renditions are pure CPU; coupling them means rendition latency inherits tagging's failures. Split by failure domain, not by step count: S3 event → EventBridge → (a) renditions function, (b) tagging function, each with its own SQS buffer, DLQ, and retry policy. Rejected: Step Functions orchestrating rendition→tag→notify — considered because "it's a pipeline," rejected because steps are independent (no cross-step state, no compensation); a state machine would add per-transition cost and a false sequential dependency. Choreography wins here. But "notify when *all* renditions + tags done" needs aggregation — that's genuinely stateful: a DynamoDB counter with conditional update (`completed_parts = expected`) fires the notification; idempotent because conditional. Considered Step Functions just for the join; kept the counter because it's one item and one condition — a state machine for a single join is machinery without payoff. (If the flow grows compensation or deadlines, revisit — that's the trigger to orchestrate.)

Duplicates: S3 events are at-least-once, and users re-upload the same file. Idempotency key = `object_etag + rendition_size`, conditional write before processing. Cold starts: all async — explicitly ignored. The rendition function uses a container image (imagemagick/libvips deps) — size it by profiling memory; Lambda CPU scales with memory, so for CPU-bound image work, *more memory is often cheaper* (finishes proportionally faster; same GB-s, lower wall time). Notify step calls the existing Rails API — wrap with retries + circuit breaker, DLQ the failures; never let the legacy system's downtime back up into the pipeline.

## Failure modes & pitfalls

- **No DLQ, FIFO/stream source, one poison message** → entire shard/group stalls, backlog grows silently until an SLA blows. DLQ + `bisectBatchOnFunctionError` (streams) + alarm on iterator age.
- **Retry × fan-out multiplication:** SNS → 3 SQS queues → functions, each layer with 3 retries → one bad event executes up to 27 times, and the downstream API you call isn't idempotent. Count *end-to-end* retry amplification for every effect with a side channel (emails, charges).
- **Timeout inversion:** Lambda timeout 60s behind API Gateway's 29s cap → gateway 504s, function keeps running, client retries, work duplicates. Function timeout must be shorter than every upstream timeout.
- **Lambda-in-VPC to RDS without RDS Proxy** → connection exhaustion at first burst. Proxy, or DynamoDB, or Aurora Data API.
- **Orchestrator code doing I/O in Durable Functions / relying on `Context.now` vs `datetime.now` confusion in replay** — deterministic-replay violations. All effects in activities; use the context's time/GUID APIs.
- **Concurrency limits as shared blast radius:** account/regional concurrency is shared; a runaway consumer starves the checkout function. Reserve concurrency for critical functions; cap it for bulk consumers (also your DB-protection throttle).
- **Payloads passed by value through the pipeline** hit the 256KB (SQS/Step Functions) limits mid-incident. Pass references (S3 key + version) between steps from day one — the claim-check pattern.
- **Warmer crons in 2026 designs** — superseded; see cold-start ladder. Their presence signals stale patterns; audit the rest of the design accordingly.
- **Emulator-green, prod-red:** IAM denied, event envelope mismatch (`Records[0].body` is a *string* containing JSON, double-encoded through SNS→SQS unless `RawMessageDelivery` is on). Test with recorded real payloads; enable raw delivery on SNS→SQS subscriptions.
- **Recursive invocation loops:** S3 event → Lambda writes a processed file *to the same bucket/prefix* → triggers itself → runaway concurrency and a five-figure bill overnight. Separate input/output prefixes and scope event filters; AWS's recursive-loop detection catches some but not all topologies (e.g., loops through SNS→SQS chains) — design it out.
- **Event source mapping batch settings fighting the timeout:** batch size 100 × 2s per record vs a 60s function timeout → perpetual partial-batch failures. Batch size × p99-per-record must fit inside the timeout with margin, or use partial-batch responses and smaller batches.
- **EventBridge rule targets failing silently:** rules have retry then drop unless a DLQ is configured *per target*. An unsubscribed-to failure mode: the bus accepted the event, the target never ran, nothing alarmed. Set target DLQs and alarm on them like queue DLQs.
- **Idempotency TTL shorter than the retry horizon:** a 1-hour idempotency window vs a DLQ redrive that happens next morning → the redrive re-executes side effects. The idempotency record must outlive the longest possible redelivery path, including human-driven redrives.
- **Cost surprise at success:** per-invocation pricing that was $50/mo at launch scaling linearly to $15k/mo at 300× traffic while a container fleet would have flattened. Revisit the FaaS-vs-container arithmetic at every order of magnitude — the right answer changes.

## Worked micro-example — idempotent SQS consumer (Python, Lambda Powertools)

```python
from aws_lambda_powertools.utilities.idempotency import (
    idempotent_function, DynamoDBPersistenceLayer, IdempotencyConfig)
from aws_lambda_powertools.utilities.batch import BatchProcessor, EventType, process_partial_response

persistence = DynamoDBPersistenceLayer(table_name="idempotency")
config = IdempotencyConfig(event_key_jmespath="orderId", expires_after_seconds=6*3600)
processor = BatchProcessor(event=EventType.SQS)

@idempotent_function(data_keyword_argument="order", config=config, persistence_store=persistence)
def handle_order(order: dict):
    charge(order)              # runs at most once per orderId within the window

def record_handler(record):
    handle_order(order=json.loads(record.body))

def handler(event, context):   # partial-batch response: only failed records return to the queue
    return process_partial_response(event=event, record_handler=record_handler,
                                    processor=processor, context=context)
```
Partial-batch response (`ReportBatchItemFailures`) matters: without it, one failure re-delivers the whole batch — idempotency then saves you, but you're relying on the seatbelt instead of not crashing.

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
The judgment encoded: retries with backoff+jitter live in the state machine (visible, tunable, no redeploy), *not* inside handler code; the timeout is explicit and shorter than the caller's; the catch route leads to a named compensation state — the saga's unhappy path is a first-class, testable part of the definition rather than an exception handler someone hopes fires.

## Verification / self-check

1. Every async edge: DLQ present, redrive path exists, alarm on depth/iterator age.
2. Every handler answers "what happens when this exact event arrives twice?" with a mechanism, not a hope.
3. Timeouts strictly decrease downstream; end-to-end retry amplification counted for every side effect.
4. Payload sizes bounded by claim-check; no by-value blobs in queues.
5. Cost computed at 1× and 10× traffic; the FaaS-vs-container crossover located.
6. At least one test tier runs against real cloud resources with recorded event shapes.
Stopping rule: when the failure path of each edge is designed (not just the happy path) and a poison message demonstrably ends in a DLQ with an alarm, stop hardening — additional resilience patterns beyond that are speculative complexity.
