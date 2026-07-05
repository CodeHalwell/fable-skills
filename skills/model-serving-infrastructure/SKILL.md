---
name: model-serving-infrastructure
description: Load when designing, costing, or debugging production model inference infrastructure — choosing managed API vs serverless GPU vs dedicated cluster, GPU utilization economics and break-even math, autoscaling and cold starts, token streaming/SSE and retry semantics, multi-model and LoRA serving, model rollout, or serving observability (TTFT, queue depth, KV cache).
---

# Model Serving Infrastructure

## Core mental model

- **You buy GPU-time; you sell tokens (or predictions). Utilization is the exchange rate.**
  A GPU bills per second whether saturated or idle. Every architecture decision — batching, autoscaling,
  fractional sharing, serverless — is a strategy for closing the gap between GPU-seconds paid and
  GPU-seconds doing useful work. Evaluate every serving proposal by asking "what does this do to
  sustained utilization?" before anything else.
- **LLM inference has two phases with opposite bottlenecks.**
  Prefill (processing the prompt) is compute-bound and parallel — an H100-class GPU chews through prompt
  tokens at tens of thousands per second. Decode (generating output) is memory-bandwidth-bound and
  sequential — hundreds of tokens/second aggregate, and only batching many concurrent requests recovers
  throughput. This asymmetry explains continuous batching, chunked prefill, prefix caching,
  prefill/decode disaggregation, and why input tokens are cheap and output tokens are expensive.
  When something about LLM serving seems weird, map it back to this asymmetry.
- **Latency and utilization are the same knob viewed from opposite sides.**
  Bigger batches raise throughput per GPU and raise per-request latency (inter-token latency degrades as
  batch grows; queueing delays TTFT). No setting maximizes both. Define the SLO first (e.g., p95 TTFT
  < 800ms, p95 inter-token < 60ms), then push batch size until the SLO is at the edge — utilization
  beyond that point is bought with SLO violations. "Goodput" (throughput of requests meeting SLO) is the
  metric, not raw tokens/sec.
- **Cold start is the tax on scale-to-zero.**
  Serverless pricing looks magical until the first request after idle pays 10–120s to pull an image and
  load 10–140GB of weights. Every mitigation (snapshotting, weight streaming, warm pools) is spending
  money or engineering to shrink that tax. Decide scale-to-zero per endpoint by comparing the tax
  (cold starts/day × latency cost) to the idle burn saved.
- **Generation is expensive and non-idempotent by default; retries multiply cost.**
  A retried 5-cent, 90-second LLM call is not a retried 0.1ms Redis GET. Every retry policy, timeout,
  and load-balancer default inherited from web-service practice must be re-examined when a single
  request costs cents and runs for minutes.

## Decision framework: managed API vs serverless GPU vs dedicated

| | Managed per-token API | Serverless GPU | Dedicated GPUs |
|---|---|---|---|
| Cost shape | Pure per-token; zero at zero traffic | Per-second while busy, ~1.5–3× on-demand rate (2026 ballpark) | Per-hour always; reserved −30–40% |
| Wins when | Commodity open weights, any duty cycle; spiky custom traffic below break-even utilization | Custom weights, duty cycle roughly <40–50% | Custom weights, sustained high duty cycle |
| Cold start | None (provider's problem) | Yours to mitigate: ~5–15s small models w/ snapshotting, 30–90s big models (as of 2026) | None once warm; capacity planning instead |
| You operate | Nothing | Container + engine config | Everything: nodes, engine, scaling, rollout |
| Examples (alive as of mid-2026) | Fireworks, Together, cloud-hosted equivalents | Modal, RunPod, Baseten, Replicate (Cloudflare-owned since early 2026) | Specialized clouds (~$2–3.3/hr H100), hyperscalers at 2–5× that |

Ask these questions in order; each answer prunes the tree:

1. **Are the weights custom (fine-tune, proprietary), or commodity open weights?**
   Commodity open weights → default to a managed per-token API. Providers run at higher utilization than
   you ever will and pass some back: as of mid-2026, Llama-3.3-70B-class inference is ~$0.90/M tokens.
   Self-hosting rarely beats that on raw price (see worked example 1). Leave managed only for: custom
   weights they won't host cheaply, data-residency/compliance, latency SLOs they won't sign, or volumes
   where the arithmetic flips.
2. **If custom weights: what's the traffic shape?**
   Compute the duty cycle: busy GPU-seconds ÷ wall-clock seconds over a representative week. Spiky or
   diurnal with long idle valleys → serverless. Sustained load → dedicated with reserved pricing.
   The whole comparison is (serverless premium × duty cycle) vs (1.0 × always-on) — worked example 2.
3. **If serverless: can you tolerate its cold start for THIS model size?**
   As of 2026: sub-second cold starts are real only for small containers (RunPod's FlashBoot claims
   sub-200ms for a large share of requests on small images). Large-model reality: Modal's GPU memory
   snapshotting brought a vLLM 7B-class server from ~2 minutes to ~10s (single-digit seconds best case);
   Baseten-class platforms still see 30–90s for big model containers after scale-down. If the product
   can never show a 30s stall, you need a warm floor (min replicas ≥ 1) — which erodes the serverless
   cost story; recompute step 2 with the floor priced in.
4. **If dedicated: does one node hold the model?**
   Fits on one GPU → plain vLLM behind a standard load balancer; add no orchestration you don't need.
   Fits on one node → vLLM with tensor parallelism, still simple. Needs multi-node, disaggregated
   prefill/decode, or fleet-scale KV-cache-aware routing → the 2026 distributed layer: llm-d or KServe
   on Kubernetes (NVIDIA GPU Operator underneath), NVIDIA Dynamo (1.0 GA as of March 2026) for
   disaggregated serving, Ray Serve when multiplexing many models on one cluster. Adopt these for a
   measured reason (routing losses, cache-hit rates, cross-node models) — not because they're modern.
5. **What would change the answer?**
   Traffic 5×-ing (serverless premium starts to dominate), the provider deprecating your model, a new
   fine-tune requirement, or per-token prices moving — they fall steadily; re-run the break-even
   quarterly and date every cached calculation.

## GPU utilization economics

- **Batch to saturate decode.** A single-request decode uses a few percent of an H100's capability.
  Continuous batching (vLLM default) turns 20 tok/s into hundreds aggregate. If per-GPU throughput is
  far below published benchmarks for your model class, the cause is almost always insufficient
  concurrency, `--max-num-seqs` too low, or KV-cache memory eaten by long contexts — not the engine.
- **Right-size the GPU to the model, then fill the leftover.** A 7B model at FP8 uses ~8GB of an 80GB
  GPU; the rest is KV cache (good — more batch) or waste (you bought an H100 for traffic an L4 could
  hold). For small models, several cheap GPUs usually beat one big one on $/token.
- **Fractional GPUs (as of 2026):**
  - **MIG** (Ampere and later): partitions one GPU into up to 7 hardware-isolated instances with
    dedicated compute and memory. Use for co-locating small latency-sensitive models where noisy
    neighbors are unacceptable.
  - **Time-slicing**: sharing with no isolation — one tenant's long kernel stalls everyone. Fine for
    dev and bursty batch; never behind a latency SLO.
  - **MPS**: between the two — concurrent kernels, no memory isolation.
  - On Kubernetes, allocation is moving from the static device plugin to **DRA** (Dynamic Resource
    Allocation); NVIDIA's DRA driver was donated to the CNCF in 2026 and is the direction of travel.
    But MIG shapes still require static pre-configuration — DRA does not yet re-partition dynamically
    (as of mid-2026). Practical consequence: choose MIG geometry per node pool ahead of time and label
    pools by shape.
- **Fractional GPUs are irrelevant for big LLMs.** A model that fills the GPU can't share it.
  Fractionalization is for the long tail — embedders, rerankers, classifiers, ASR — where it routinely
  triples effective utilization.

## Autoscaling for inference

- **Scale on work waiting, never on CPU or GPU "utilization %".** CPU idles on a GPU server; nvidia-smi
  GPU-util reads high even when memory-stalled at low throughput — both lie. Correct signals, in
  preference order: queue depth (admitted but not started), in-flight requests per replica vs a
  measured max, KV-cache utilization. KServe/Knative concurrency targets and KEDA-on-queue-length are
  the standard mechanisms; vLLM exposes the signals via Prometheus (`vllm:num_requests_waiting`,
  KV-cache usage gauges) — wire the autoscaler to those.
- **Set the scale-up threshold from measured cold-start time, not taste.** If a replica takes 90s to
  serve, scale when the queue predicts saturation 90s out. If traffic can spike faster than replicas
  boot, no threshold saves you — you need a warm pool or standing headroom (~20–30% is typical for
  spiky LLM traffic).
- **Cold-start mitigation ladder** (cheapest first):
  1. Bake weights into the image or a pre-warmed volume — never download from a remote hub at boot.
  2. Stream weights from region-local object storage, overlapping download and GPU load
     (safetensors + parallel range reads; managed platforms do this for you).
  3. Snapshot/restore of initialized GPU state (Modal's GPU memory snapshotting class of technique,
     as of 2026).
  4. Warm pools — booted, weights-loaded replicas held out of rotation. You pay idle for insurance;
     size the pool to spike statistics, not the worst case ever seen.
- **Scale-to-zero is a product decision.** Zero replicas means the next user waits 10–90s. Right for
  internal tools, batch endpoints, long-tail models; wrong for anything interactive with steady daytime
  traffic. The compromise: floor of 1 during business hours, zero overnight — a cron-shaped
  `minReplicas` schedule, embarrassingly effective.

## Streaming APIs done right

- **Use SSE (`text/event-stream`); it's the ecosystem convention** (OpenAI-compatible APIs, vLLM's
  server). Event per chunk, terminal `data: [DONE]` sentinel, heartbeat comments (`: ping`) every ~15s
  so long prefills don't trip proxy idle timeouts. Disable buffering explicitly on every hop
  (`X-Accel-Buffering: no`; nginx `proxy_buffering off`) — one buffering ALB/CDN layer turns the stream
  into a single blob delivered at completion.
- **Timeout per gap, not per request.** A 4k-token generation legitimately runs minutes. Set (a) a TTFT
  timeout (~30–60s — nothing arrived, request is queued or stuck, safe to act) and (b) an inter-chunk
  timeout (~10–30s — the stream died). A single whole-request timeout either kills legitimate long
  generations or lets zombies run forever.
- **Retries must be idempotent or they double the bill.** Naive client retry on timeout while the
  server still generates = two GPUs computing the same answer, possibly two answers delivered.
  Require client-supplied idempotency keys; on retry, reattach to the in-flight generation (or replay
  from a result cache) rather than restarting. Auto-retry only errors that provably occurred before
  generation started (connect failures, 429/503 at admission). Never auto-retry mid-stream failures at
  the infra layer — surface them with the partial output and let the client decide.
- **Backpressure: the GPU produces tokens whether or not anyone reads them.** A slow client on an
  unbounded server buffer accumulates memory; a *disconnected* client whose request keeps generating
  burns GPU for no one. Propagate disconnect to the engine as an abort (vLLM supports request abort;
  verify your HTTP layer actually forwards cancellation — many async frameworks silently don't).
  Cap per-connection send buffers; abort on sustained non-consumption.

## Multi-model serving

- **Route at a gateway; multiplex on shared GPUs only for small models.** One process per big model;
  an OpenAI-compatible gateway maps the `model` field to a backend pool. Co-locating two big LLMs on
  one GPU halves each one's KV cache and couples their failure modes — don't, unless both are small
  and MIG-isolated.
- **Multi-LoRA is the exception that changes the economics.** Dozens-to-hundreds of fine-tunes sharing
  one base can be served from a single engine: base weights loaded once, per-request adapter selection,
  adapter-aware batching. vLLM has native multi-LoRA (`--enable-lora`, adapter name in the request's
  `model` field); LoRAX established the pattern. Adapter hot-load from local disk is
  milliseconds-to-subsecond for low ranks (as of 2026). Costs: small per-token overhead; adapters must
  share the exact base (same version, same tokenizer); adapter-cache thrash if the working set exceeds
  GPU headroom. If you fine-tune per customer, design for LoRA-on-shared-base from day one —
  full-weights-per-customer is a 10–100× infra cost mistake.
- **Fleet-scale routing (as of 2026):** KV-cache-aware routing — send a request to the replica already
  holding its prefix (same session, same system prompt) — is productized in llm-d, NVIDIA Dynamo, and
  AIBrix; prefill/decode disaggregation (separate prefill and decode worker pools with KV transfer)
  lives in the same layer. Both matter when prefixes are long and reuse is high (agents, chat with big
  system prompts): a routed cache hit skips most of prefill. Below fleet scale, get 80% of the value
  with session-sticky routing (hash session ID → replica) plus vLLM's automatic prefix caching within
  each replica.

## Versioning and rollout for models

- **Immutable versions, pinned deploys, no "latest".** A model version = weights hash + engine version
  + sampling defaults + prompt template. Engine upgrades are model changes: the same weights on a new
  engine version can change outputs — canary engine bumps like model bumps.
- **Rollout ladder: offline eval gate → shadow → canary → progressive.** The eval gate (fixed golden
  set, task metric or judged comparison vs incumbent) is what makes automated rollout safe; without it
  you ship vibes. Shadow (duplicate real traffic, log candidate outputs, act on nothing) validates
  latency, OOM behavior, and output-distribution sanity under real prompts — it cannot measure user
  impact. Canary by *user/session*, not by request: per-request assignment gives one user alternating
  model personalities mid-conversation and contaminates measurement.
- **Keep N and N−1 warm during rollout.** Model rollback is not a config flip if the old weights take
  5 minutes to load; hold the incumbent's replicas until the canary clears its bake window.

## Observability specifics

Instrument these or fly blind (all exposed by vLLM-class servers via Prometheus):
- **TTFT** (p50/p95/p99) — queueing + prefill; the first metric users feel. Rising TTFT at flat traffic
  = growing queue = you're saturated.
- **Inter-token latency / per-request output tokens-sec** — decode health; degrades as batch grows;
  this is the SLO-vs-utilization dial readout.
- **Queue depth / requests waiting** — the autoscaling signal and earliest saturation alarm.
- **KV-cache utilization and preemption counts** — cache near 100% with preemptions means requests are
  being paused and recomputed; throughput falls off a cliff *before* OOM. Alert at ~90%.
- **Prefix-cache hit rate** — a regression here (e.g., after a prompt change broke prefix stability)
  silently doubles prefill cost.
- **Tokens in/out distributions per endpoint** — shape changes are cost changes; a p99 input-length
  jump predicts OOM and head-of-line incidents before they happen.
- **Quality drift**: continuously sample production outputs into an eval pipeline (judge model or task
  metric) per model version. Serving metrics can be pristine while a bad rollout, quantization change,
  or prompt regression tanks quality — output-quality sampling is the only detector.

## How an expert thinks through this

Scenario: a team fine-tuned an 8B model for support-ticket drafting. Traffic ~40k requests/day,
9-to-5 diurnal, near-zero nights and weekends. Each request ~1,500 tokens in / 300 out. They ask for
"a Kubernetes GPU cluster with KServe, llm-d, and autoscaling."

- First: is the requested stack proportionate? 40k req/day peaks at maybe 2–3 req/s. An 8B model at FP8
  on one L40S/A100-class GPU with continuous batching handles that with room to spare. This is a
  1–3 GPU problem, not a platform problem. *Reject the k8s + llm-d buildout*: disaggregation and
  KV-aware routing earn their complexity at fleet scale with long shared prefixes; here it's months of
  platform work to serve 3 req/s.
- Managed API instead? No — custom fine-tune plus ticket-data residency constraints. Note the ceiling
  anyway: at 2026 open-model prices this workload would cost roughly $65/day on a per-token API —
  a useful number to beat.
- Traffic shape: ~10 busy hours of 24, weekends off → duty cycle ~30%. Serverless zone. *Consider
  dedicated anyway*: one on-demand L40S/A100 at ~$1–2/hr is ~$30–50/day always-on — cheap enough that
  cost alone doesn't decide. Tie-breakers: ops burden, and the second GPU — peak needs burst to 2
  replicas, and owning an always-on second GPU for 2 peak hours/day is pure waste. Serverless with a
  business-hours floor of 1, queue-depth burst to 2–3, zero on weekends captures all of it.
- Cold-start check: 8B FP8 ≈ 9GB of weights; on a platform with weight caching/snapshotting that's
  ~5–15s (as of 2026). Weekend scale-to-zero means Monday's first user waits ~10s — fine for an
  internal drafting tool. *Reject weekday-overnight scale-to-zero*: a 7am early user paying a cold
  start every single morning is a daily papercut; schedule the floor to rise at 7am instead.
- Streaming: drafts are read as they generate → SSE, and disconnect-abort matters because agents close
  tickets mid-generation constantly; without abort propagation, 10–20% of GPU time goes to generating
  into closed sockets.
- Rollout: they retrain monthly. Wire the eval gate now (golden set of ~200 tickets judged against the
  incumbent), shadow a day, canary by agent ID. *Reject shadow-only promotion*: shadow can't see that
  agents edit the new model's drafts twice as much — that needs the canary plus an edit-distance metric.
- Stopping rule: one serverless endpoint, scheduled floor, queue-depth autoscaling, SSE with aborts,
  eval-gated monthly rollout. No k8s, no MIG, no router. Revisit when traffic 10×'s or a second model
  appears.

## Failure modes & pitfalls

- **OOM on long context, weeks after launch.** Tuned with `--gpu-memory-utilization 0.9` and typical
  2k-token prompts; one day a user pastes a 100k-token document and the request OOMs the engine or
  (vLLM) forces mass preemption that craters everyone's latency. Corrections: set `--max-model-len` to
  what you actually support, not the model's maximum; validate and reject/truncate over-limit inputs at
  the gateway; load-test at max context × max concurrency, not at typical values; watch preemption
  counters — preemption storms precede OOM.
- **Autoscaling on GPU utilization %.** `DCGM_FI_DEV_GPU_UTIL` reads ~95% whenever kernels are
  resident, including memory-stalled low-throughput decode — so the autoscaler thinks
  "saturated at low load" and never scales down; meanwhile a default CPU-based HPA never scales up at
  all. Correction: HPA/KEDA on `vllm:num_requests_waiting` or in-flight requests per replica.
- **Thundering herd on scale-up.** Queue builds → autoscaler adds 4 replicas → all 4 pull the same
  30GB image and weights through the same registry/NFS path → 15-minute boot instead of 2 → queue grows
  → autoscaler adds more. Corrections: region-local weight cache or pre-baked images; jittered replica
  starts; cap scale-up step size; alarm on divergence between replicas-requested and replicas-Ready.
- **Head-of-line blocking by long prompts.** Without chunked prefill, one 80k-token prefill
  monopolizes the GPU for seconds while every in-flight decode stalls — users see mid-stream freezes.
  Corrections: chunked prefill (default in vLLM's V1 engine as of 2026 — verify on your version);
  cap `--max-num-batched-tokens`; segregate a long-context pool behind the router so whale requests
  can't queue in the interactive pool.
- **Retry amplification.** Gateway timeout 60s, generation takes 90s: gateway retries (two GPUs
  generating), client library retries too (four). The bill quadruples and p99 worsens because retries
  deepen the queue that caused the timeout. Corrections: idempotency keys with in-flight deduplication
  at the gateway; retries only on pre-admission failures; a retry *budget* (max ~10% of traffic may be
  retries — a circuit breaker, not a suggestion).
- **A proxy that buffers SSE.** Works locally; behind the corporate ALB/nginx/CDN, tokens arrive as one
  lump after 40s. Correction: disable response buffering and compression for `text/event-stream` on
  every hop (`X-Accel-Buffering: no`, `proxy_buffering off`); test streaming through the full
  production path, never just localhost.
- **Client disconnects don't cancel generation.** Async HTTP frameworks often keep the handler running
  after the socket closes; the engine completes the generation for nobody. At chat/agent abandonment
  rates this is 10–30% of GPU spend. Correction: verify disconnect → engine abort end-to-end — kill a
  client mid-stream and watch the running-request gauge drop within seconds.
- **Prefix cache as a cross-tenant side channel / correctness hazard.** Shared automatic prefix caching
  lets one tenant's cached prefix speed up (and via timing, reveal the existence of) another's
  identical prefix; naive custom KV caches have returned wrong-tenant state outright. Correction:
  per-tenant cache salting (vLLM supports a `cache_salt` request field, as of 2026) or per-tenant pools
  where isolation matters.
- **Breaking prefix stability with a "harmless" prompt change.** Someone prepends
  `Current time: 09:41:07` to the system prompt; every prefix is now unique, prefix-cache hit rate
  drops to ~0, prefill cost doubles, TTFT jumps, and nobody connects the dots for a week. Correction:
  volatile content goes at the *end* of the prompt; monitor hit rate and alert on step changes.
- **The MIG shape trap.** MIG partitions are fixed geometries chosen per GPU ahead of time; a node
  pre-carved into 7 small slices cannot serve a new 13B model without draining and re-partitioning
  (dynamic MIG reconfiguration is still not handled by the DRA driver, as of mid-2026). Correction:
  separate node pools per MIG geometry; keep whole-GPU nodes in reserve; treat re-partitioning as a
  node-lifecycle operation.
- **Judging cost by output tokens only, or by list price only.** Self-host vs API comparisons fail both
  ways: forgetting APIs charge for input tokens (input-heavy workloads favor self-hosting far more than
  output-token math suggests), and forgetting your dedicated GPUs run at 30% utilization while the
  provider's run at 70%+ (favoring the API). Do the arithmetic with your real request shape and
  measured duty cycle — worked example 1.
- **Shadow-mode conclusions about quality.** Shadow validates latency, stability, and output
  *distributions*; it cannot tell you users prefer the new model, because nothing acted on its outputs.
  Promoting on "shadow looked fine" skips the only measurement that matters.
- **`max_tokens` left at a huge default.** Runaway generations (repetition loops) run to the 8k cap,
  holding batch slots and KV memory for minutes. Correction: per-endpoint `max_tokens` matched to the
  product (a ticket draft never needs 8k), stop sequences or repetition penalties, and an alert on p99
  output length.
- **Warm-up not exercised before rotation.** A replica marked Ready after the HTTP port opens but
  before CUDA graphs are captured and the first batch compiled serves its first requests 5–10× slow,
  dragging p99 on every scale-up. Correction: readiness probe fires only after a synthetic warm-up
  generation completes.

## Worked micro-examples

**1. Break-even: managed API vs dedicated H100 (2026 ballparks — re-verify prices before relying on this).**
Workload: 70B-class model, requests of 2,000 tokens in / 400 out.
- Managed API: ~$0.90/M tokens (Llama-3.3-70B-class on Fireworks/Together, mid-2026, flat in/out).
  Per request: 2,400 tok × $0.90/M ≈ **$0.00216**.
- Dedicated: H100 SXM ~$2.50/hr (specialized clouds ran ~$2.0–3.3/hr on-demand in 2026; hyperscalers
  2–5× that) = $0.000694/GPU-s. Saturated per-GPU throughput for 70B at FP8: roughly 450–500 output
  tok/s aggregate at high batch; prefill ~15k tok/s (2026-era published benchmarks; verify for your
  engine and quantization). GPU-time per request at saturation ≈ 400/460 + 2,000/15,000 ≈ 1.0 GPU-s →
  **$0.00069 at 100% utilization**.
- Break-even utilization = 0.00069 / 0.00216 ≈ **32%**. Below ~32% sustained duty cycle the API is
  cheaper; above it, self-hosting wins — ~3× cheaper at full saturation. One saturated H100 handles
  ~86k such requests/day (~207M tokens/day); break-even is ~28k req/day (~66M tokens/day) on this shape.
- Sensitivity: output-heavier shapes push break-even *up* (decode dominates GPU time while the API
  bills all tokens equally); reserved pricing (−30–40%) and high prefix-cache hit rates push it *down*.
  Single-region diurnal traffic won't sustain 32% without scale-to-zero — which is exactly the
  serverless pitch.

**2. Serverless vs always-on, same GPU.** Serverless H100-class runs ~1.5–3× the on-demand per-second
rate (2026 ballpark; take 2× → $5/hr-equivalent vs $2.50/hr dedicated). Serverless cost = $5 ×
busy-hours; dedicated = $2.50 × 24 = $60/day. Break-even busy time = 60 / 5 = **12 h/day** at the 2×
premium (8 h/day at 3×). A 9-to-5 workload (~8–10 busy hours) sits right at the line — which is why the
hybrid (a floor replica at dedicated-style pricing + serverless burst) usually beats either pure option.

**3. vLLM flags that encode the above** (names stable as of 2026; confirm with `vllm serve --help` on
your version):
```bash
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --tensor-parallel-size 2 \
  --max-model-len 16384 \            # what you support, not what the model allows
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 128 \               # batch ceiling: raise until the SLO edge, watching ITL
  --enable-lora --max-loras 8        # multi-LoRA on a shared base, if applicable
# Prefix caching and chunked prefill are on by default in the V1 engine (as of 2026).
# Autoscale on vllm:num_requests_waiting; alert on KV-cache usage >0.90 and preemptions >0.
```

## Verification / self-check

Before presenting a serving design or diagnosis, confirm:
- [ ] The cost comparison uses the *real* request shape (in:out ratio), measured or honestly estimated
      duty cycle, and prices checked this quarter — every number dated, nothing recalled from memory.
- [ ] SLO stated first (TTFT, inter-token, availability); batching and scaling derived from it — not
      "maximize utilization" in a vacuum.
- [ ] Autoscaling signal is queue/concurrency/KV-based; scale-up threshold accounts for measured
      cold-start time; the thundering-herd path (N replicas booting at once) is analyzed.
- [ ] Streaming tested through the full proxy chain; disconnect-abort verified by killing a client
      mid-stream; retry policy idempotent with a retry budget.
- [ ] Long-input behavior load-tested (max context × concurrency); gateway input caps and per-endpoint
      `max_tokens` bounds in place.
- [ ] Rollout has an offline eval gate and an entity-randomized canary; rollback keeps the incumbent
      warm.
- [ ] Complexity is proportionate: every stack component (k8s, disaggregation, KV-aware routing, MIG)
      is justified by a number, not by fashion.

Stopping rule: the design is done when the SLO is met at peak with ~20–30% headroom, the cost model
beats the best alternative you rejected (and you can show the arithmetic), and the failure drills —
cold start, traffic spike, long prompt, mid-stream disconnect, bad-model rollback — have each been run
once. Optimization beyond that is speculative; revisit when traffic shape, model size, or prices change
materially.
