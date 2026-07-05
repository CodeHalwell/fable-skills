---
name: llm-observability
description: Load when instrumenting, monitoring, or debugging LLM applications in production — tracing prompts/completions/tool calls, choosing observability tooling, designing online evaluation and feedback capture, drift and regression detection, cost/latency dashboards, quality alerting, or working from a bad output back to root cause.
---

# LLM Observability in Production

## Core mental model

- **The trace is the unit of debugging.** Not the log line, not the request metric — the full tree: user input → system prompt version → retrieval calls with results → each model call with exact rendered prompt, parameters, completion, token counts → tool calls with arguments and results → final output, all correlated under one trace ID. If you can't replay what the model actually saw, every production bug report is unanswerable. Instrument this before launch, not after the first incident.
- **LLM failures are silent.** A 200 response containing a hallucination, a refusal, or a truncated answer looks identical to success in conventional APM. Availability monitoring tells you almost nothing; you need *quality* signals flowing continuously from production.
- **Users are your cheapest evaluators — implicitly.** Explicit thumbs get 0.1–1% response rates and skew negative. Implicit signals cover everything: did the user edit the output, regenerate, copy it, rephrase and retry, abandon the session? These are behavioral labels on every interaction. Design the product to emit them.
- **Every model, prompt, or retrieval change is a deploy** and deserves the same gate as a code deploy: run the offline eval suite, then canary against online metrics. Most "sudden quality drops" trace to an unversioned prompt edit or a provider-side model update nobody pinned.
- **Observability closes the loop or it's decoration.** The point of traces is trace *mining*: bad traces → failure clusters → new eval cases → fixes → verified on the same dashboards. A team whose eval set doesn't grow from production traces has observability theater.

## Tooling and standards (verified as of mid-2026)

- **OpenTelemetry GenAI semantic conventions** are the emerging wire standard: `gen_ai.*` attributes (`gen_ai.request.model`, `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`), spans for model/tool/agent operations, with full prompt/completion content capture as an explicit **opt-in**. Status: still in Development (not stable) as of mid-2026 — attribute names may shift; use `OTEL_SEMCONV_STABILITY_OPT_IN` dual-emission when upgrading instrumentation, and don't hand-roll attribute names when an instrumentation library exists.
- Platform classes, choose by constraint, not fashion:
  - **Langfuse** — MIT open source, self-hostable, OTel-native, ClickHouse-backed (acquired by ClickHouse, Jan 2026). Default when data residency/self-hosting matters or you want vendor neutrality.
  - **LangSmith** — managed, deepest LangChain/LangGraph integration. Default if you're already committed to that framework.
  - **Braintrust** — proprietary, eval-first: eval scores native to trace views, CI quality gates. Default when evaluation workflow is the center of gravity and budget allows.
  - Rolling your own on raw OTel + ClickHouse/Datadog is viable for the *transport*, but you'll rebuild trace-tree UIs, eval integration, and prompt-diff views — usually a losing trade below large scale.
- Whichever you pick, keep instrumentation OTel-shaped so the vendor is swappable.

## What to log — and what not to

Reasoning chain: *Can I store full payloads?* → check privacy class (PII? regulated?) and volume cost. Then:

- **Metadata for 100% of traffic, always**: trace structure, model + version, prompt template ID + version, token counts, latencies per span, finish reasons, tool-call names and status, retrieval doc IDs and scores, user/session IDs (pseudonymous), feedback events. This is cheap and powers every dashboard.
- **Full content (prompts/completions/tool payloads) sampled**: 100% while traffic is small (the traces you need are always the ones you didn't keep); as volume grows, keep 100% of *interesting* traces (errors, negative feedback, regenerations, judge-flagged, long-latency, canary traffic) + a fixed random sample (1–10%) for unbiased mining. Tail-based sampling — decide after the trace completes — not head-based, or you'll sample away exactly the failures.
- **PII redaction runs before the trace leaves your process** (presidio-style scrubbing on content fields), not in the observability backend. Redact user content, keep structure. Retention: content days-to-weeks, metadata months-to-years.
- Never log: raw credentials in tool arguments (scrub known-sensitive tool params by allowlist), full retrieval corpus chunks when doc IDs suffice for replay against a versioned index.

## Online evaluation architecture

Three layers, cheapest first:

1. **Implicit behavioral signals** (100% coverage, free): regeneration = strong negative; substantial user edit of output = negative with a diff that tells you *what* was wrong; copy/insert/accept = weak positive; immediate rephrase-and-retry = the model missed intent; session abandonment after response = weak negative. Log these as first-class feedback events attached to the trace ID.
2. **Explicit feedback** (sparse, biased negative — calibrate, don't trust rates as absolute quality): thumbs, categorized reports. Value is in the *attached trace*, not the aggregate rate.
3. **Sampled LLM-judge scoring on production traffic**: run binary-rubric judges (grounded-in-context? answered-the-question? refused?) on a few percent of traces, async, off the request path. Pin the judge model + prompt version; recalibrate against human labels when either changes. Judge scores are for *trends and triage*, not per-user decisions.

Wiring the loop end to end — the **feedback-to-improvement loop** as a standing process, not an aspiration:
1. Weekly: pull the union of (negative implicit signals, explicit reports, judge-flagged traces) for the window.
2. Cluster them — embedding the user inputs and k-means/HDBSCAN-ing is enough; you're looking for the 3–5 dominant failure themes, not a taxonomy.
3. For each cluster: read 5–10 traces, run the root-cause tree (below), and promote 2–3 representative cases into the offline eval set *before* attempting the fix — the eval case is what proves the fix and prevents regression.
4. Ship the fix through the eval-on-deploy gate; verify the cluster's judge-score/feedback trend actually moves. A fix that doesn't move the dashboard didn't fix the cluster.
Track one meta-metric: eval cases added from production per month. Zero means the loop is broken regardless of how good the dashboards look.

## Agent-specific observability

Agents multiply everything: one user request = N model calls, M tool calls, loops, and self-corrections. Additions beyond basic tracing:
- **Span hierarchy must mirror the agent graph**: agent step → model call → tool call(s), with parent-child links intact. A flat list of 40 spans for one request is unreadable at incident time.
- **Loop/budget telemetry as first-class attributes**: steps taken, tokens consumed vs. per-trace budget, distinct-tools-used, repeated-tool-call count (same tool + same args twice = warning sign; three times = loop).
- **Tool error vs. tool-error-handled distinction**: log whether the model acknowledged a failed tool result or narrated past it — the latter is the dangerous one, and it's detectable (tool span status=error followed by a confident final answer with no retry).
- **Trajectory outcome labels**: completed / gave-up / budget-cutoff / user-abandoned. The cutoff and gave-up rates are the agent's real reliability metrics; final-answer judge scores alone miss the requests that never produced an answer.

## Drift and regression detection

- **Input drift**: monitor input length distribution, language mix, topic cluster shares (embed + cluster daily, compare to baseline), and rate of out-of-scope requests. Input drift explains quality drops that no deploy caused — your users changed, or a new integration started sending garbage.
- **Output drift**: refusal rate, output length, format-parse failure rate (for structured outputs), judge-score trend. A step change in output length or refusal rate with no deploy on your side almost always means the *provider* changed the model under an unpinned alias — pin model versions; treat provider model updates as deploys you must gate.
- **The eval-on-deploy gate**: any change to prompt, model, retrieval config, or tool schema runs the offline eval suite in CI (block on regression), then canaries at 5–10% with judge scores + implicit signals compared against control before full rollout. Prompt edits merged without this gate are the #1 source of regressions in practice — prompts must live in version control or a versioned prompt registry, never edited live in a dashboard textbox without a linked eval run.

## Cost and latency attribution

- Attribute tokens to **feature × prompt-template × model**, not just per-API-key. One team's runaway agent loop shouldn't be discoverable only in the monthly invoice. Emit `input_tokens`, `output_tokens`, cached-token counts per span; multiply by rate cards in the dashboard layer so price changes don't require re-instrumentation.
- Watch: cost per *session* (not per call — agents multiply calls), p95 tokens per trace (catches prompt bloat and context stuffing), cache hit rate if using prompt caching, and cost-per-successful-outcome (cost divided by positive-feedback traces) as the north-star efficiency metric.
- Latency: track time-to-first-token and total generation separately; TTFT regressions are provider/queueing issues, total-time regressions are usually your own prompt growth or added chain steps.

## The debugging workflow: bad output → root cause

Decision tree an expert actually runs, in order of prior probability:

1. **Pull the trace.** Read the exact rendered prompt. ~40% of the time the bug is visible right there: template variable empty, wrong system prompt version, context truncated, conversation history mangled. (Prompt bugs are the most common and least glamorous cause — check first.)
2. **Retrieval miss?** (RAG apps: next ~30%.) Were the retrieved chunks relevant? If the answer isn't in the context, the model didn't hallucinate — it was set up. Check retrieval scores, then whether the right doc exists in the index at all (ingestion gap vs. ranking failure — different fixes).
3. **Tool failure?** Did a tool return an error or empty result the model then papered over? Agents narrating success over failed tool calls is a classic; check tool spans' status and outputs, not the model's claims about them.
4. **Model regression?** Only after 1–3 are clean: did model version, temperature, or provider alias change? Replay the same trace inputs against the previous model version — traces make this a five-minute experiment.
5. **It's the model being a model**: irreducible stochastic failure. Add it to the eval set; consider prompt hardening; don't chase single-instance ghosts without a cluster.

## Alerting design

- Alert on **rates and steps, not instances**: refusal-rate spike (e.g., >3× 7-day baseline over 30 min), parse-failure rate for structured outputs, judge-score drop beyond noise band, error-*loop* detection for agents (same tool failing N times within one trace — page-worthy, it burns money in real time), TTFT p95, and cost-per-hour runaway (a misconfigured retry loop can 100× spend before a monthly budget alert fires).
- Define quality SLOs the same way as availability SLOs: "≥97% of sampled traces pass the groundedness judge, 7-day window" — with an error budget that gates risky prompt experiments.
- Every alert links to the trace sample that triggered it. An alert without traces attached is a mystery, not a signal.

## How an expert thinks through it: "quality dropped Tuesday"

Support-bot CSAT dips; no deploy in the changelog. Internal monologue: *No deploy — but the changelog only covers code. Check the prompt registry: no change. Check model pinning… the config says `gpt-x-latest`-style alias, not a pinned snapshot. Provider release notes: model updated Monday. Strong suspect — but verify, don't conclude.* Pull 50 negative-feedback traces from Tuesday vs. 50 from last week. *Diff the behavior: refusal rate similar (weakens the model-update theory for refusals), but answers now cite the wrong plan tier.* Look at retrieval spans: same doc IDs as before. *So not retrieval ranking… open the chunks themselves — they contain BOTH old and new pricing; the pricing page was re-ingested Monday with a migration that concatenated versions.* Root cause: ingestion bug, coincident with (innocent) model update. *Rejected hypotheses and why: model update (behavior diff didn't match — refusals stable, factuality on one topic broken); prompt regression (registry immutable, no change); input drift (topic mix unchanged in the input dashboard).* Fixes: re-ingest with dedup, add a judge rubric "cites exactly one plan tier" to online sampling, pin the model alias, and add the 50 bad traces as eval cases. Stopping rule: judge score for the pricing slice back to baseline for 48h, and the new eval cases pass in CI.

## Failure modes & pitfalls

- **Logging only final input/output, no intermediate spans.** When an agent misbehaves you can't tell which of nine steps went wrong. Correction: span per model call, per tool call, per retrieval — trace-tree from day one.
- **Head-based sampling** (decide at request start) drops the failures you exist to catch. Correction: tail-based — buffer, decide on error/feedback/judge flags after completion.
- **Unpinned model aliases in production.** Provider updates land as unexplained regressions. Correction: pin snapshots; upgrade deliberately through the eval gate.
- **Prompt edited in a dashboard, not versioned.** The trace says template v14 but nobody can diff v14 against v13. Correction: prompts in git or a registry with immutable versions; trace records the version ID.
- **Averaging judge scores across all traffic** hides slice failures (one language, one intent, one tool). Correction: slice dashboards by intent/feature/language; alert per-slice.
- **Treating thumbs-down rate as quality ground truth.** It measures annoyance × willingness-to-click. Correction: use it for triage and trend, calibrate against judged samples.
- **Judge drift**: judge model or rubric silently updated → dashboards show a "regression" that is actually the measuring stick changing. Correction: pin judge versions; re-baseline explicitly on judge changes; annotate dashboards with judge-version boundaries.
- **PII in traces discovered at audit time.** Redaction added "later" means months of raw user data in a third-party backend. Correction: redact client-side/in-process before export; verify with a scanner on the stored traces.
- **Cost dashboards keyed on API key only** — can't answer "which feature doubled spend." Correction: feature + template attributes on every span.
- **Agent error loops invisible until the invoice**: retries + tool failures + "let me try again" can run for minutes. Correction: per-trace step and token budget with hard cutoff, alert on cutoff rate.
- **Eval set frozen at launch** while production distribution moves. Correction: scheduled trace-mining ritual — weekly, pull the worst-judged and negative-feedback traces, cluster them (embedding clustering works), promote representative failures to eval cases. If eval-set growth is zero for a month, the loop is broken.
- **Judging only traces that already have negative signals** and reporting the judge pass-rate as "quality." That's the failure population's pass-rate. Correction: maintain the unbiased random judge sample as the headline metric; use signal-triggered judging for triage only.
- **Trace IDs that don't propagate across service boundaries** — the retrieval service, the agent loop, and the frontend feedback event each mint their own. The trace tree exists in three databases and joins nowhere. Correction: W3C trace context propagation end-to-end; the feedback widget carries the trace ID from the response headers.
- **Logging the prompt template but not the rendered prompt** (or vice versa). Template-only can't show you the bad variable substitution; rendered-only can't tell you which version to fix. Correction: template ID + version as attributes, rendered content in the sampled payload.

## Worked micro-example: OTel-shaped span for a model call

```python
# Attribute names per OTel GenAI semconv (Development status as of mid-2026 —
# prefer an instrumentation library, e.g. OpenLLMetry/OpenInference/Langfuse SDKs,
# over hand-rolling; shown expanded for clarity).
with tracer.start_as_current_span("chat claude-sonnet-4-5") as span:
    span.set_attribute("gen_ai.operation.name", "chat")
    span.set_attribute("gen_ai.request.model", REQUEST_MODEL)      # pinned version string
    span.set_attribute("app.prompt_template.id", "support_answer") # your own namespace
    span.set_attribute("app.prompt_template.version", "v14")
    resp = client.messages.create(model=REQUEST_MODEL, messages=redacted(messages), ...)
    span.set_attribute("gen_ai.usage.input_tokens", resp.usage.input_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", resp.usage.output_tokens)
    span.set_attribute("gen_ai.response.finish_reasons", [resp.stop_reason])
    if CAPTURE_CONTENT:  # opt-in, tail-sample decision recorded at trace end
        span.add_event("gen_ai.content", {"prompt": redact(messages), "completion": redact(resp)})
```

Feedback joins later by trace ID: `record_feedback(trace_id, kind="regenerate", value=-1)`.

## Worked micro-example: tail-sampling + judge-sampling policy

```python
# Decide retention AFTER the trace completes (tail-based), then decide judging.
def sampling_decision(trace) -> dict:
    interesting = (
        trace.error
        or trace.feedback in {"thumbs_down", "regenerate", "large_edit"}
        or trace.finish_reason == "max_tokens"          # truncation = quality suspect
        or trace.total_latency_ms > LATENCY_P99
        or trace.parse_failed                            # structured-output break
        or trace.deployment == "canary"                  # 100% of canary, always
    )
    keep_content = interesting or (hash(trace.id) % 100 < 5)   # + 5% random baseline
    # Judge a subset of what we keep: all interesting, 1% of random keeps.
    run_judge = interesting or (keep_content and hash(trace.id) % 500 == 0)
    return {"keep_content": keep_content, "run_judge": run_judge,
            "keep_metadata": True}                       # metadata: always, 100%
```

Two properties to preserve when adapting: the random slice exists (interesting-only sampling
biases every trend metric toward failures — you need the unbiased denominator), and canary
traffic is exempt from sampling entirely (it's the population you're gating a deploy on).

## Verification / self-check

Your observability is adequate when you can answer, each within ~5 minutes, using only the tooling:
1. "Show me the exact prompt and context the model saw for this complaint" (trace replay).
2. "Did quality change after yesterday's prompt merge?" (versioned deploy markers on judge/feedback dashboards).
3. "What are the top 3 failure clusters this week, with example traces?" (trace mining).
4. "Which feature spent the most tokens yesterday and was it useful spend?" (attribution + outcome join).
5. "Would we notice within an hour if refusals tripled or an agent started looping?" (alert test — fire a synthetic incident and check).

Stopping rule: instrument until those five queries work, then stop adding telemetry and start *consuming* it — the weekly trace-mining ritual delivers more quality improvement than any additional dashboard.
