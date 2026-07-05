---
name: llm-observability
description: Load when instrumenting, monitoring, or debugging LLM applications in production — tracing prompts/completions/tool calls, choosing observability tooling, designing online evaluation and feedback capture, drift and regression detection, cost/latency dashboards, quality alerting, or working from a bad output back to root cause.
---

# LLM Observability in Production

Assumed baseline (verified expert-grade cold): the trace tree as the debugging unit with span-per-model/tool/retrieval call; thumbs response rates 0.1–1% and negativity bias; implicit signals ranked (regenerate/large-edit strong negative, copy/accept positive, task outcome strongest); tail-based sampling with 100% of interesting traces plus a mandatory unbiased random slice as denominator; no-deploy regressions traced first to unpinned provider aliases, then input drift, then ingestion, then measurement artifacts; agent loop detection (repeated tool+args, step budgets, outcome labels); paging on objective fast signals and ticketing statistical quality signals; judge pinning, canary items, human-agreement calibration, position/verbosity/self-preference bias controls; repair/retry rate as the leading indicator for structured-output degradation; eval-cases-promoted-from-production as the loop-health meta-metric; session-level metrics (turns-to-resolution, repair/re-ask spirals); cost as a fact table keyed by feature × template × model with versioned price tables and cached-token counts.

## Standards and tooling facts (verified mid-2026 — the part worth pinning)

- **OTel GenAI semantic conventions are still in Development (not stable)**: `gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens`, `gen_ai.response.finish_reasons`; content capture is explicit opt-in and the *least* stable area. Use an instrumentation library (OpenLLMetry/OpenInference/vendor SDKs) rather than hand-rolled attribute names, and use `OTEL_SEMCONV_STABILITY_OPT_IN` dual-emission when upgrading — attribute renames are still landing.
- **Langfuse**: MIT, self-hostable, OTel-native, ClickHouse-backed — **acquired by ClickHouse, Jan 2026**; default for data residency/vendor neutrality. **LangSmith**: managed, deepest LangChain/LangGraph tie. **Braintrust**: proprietary eval-first with CI quality gates. Keep instrumentation OTel-shaped so the vendor stays swappable; rolling your own transport is viable but you'll rebuild trace-tree UIs and prompt-diff views.

## Discipline rules

- Every model/prompt/retrieval/tool change is a deploy: offline eval suite in CI (block on regression) → 5–10% canary judged against control → rollout. Prompts live in version control or an immutable registry; a dashboard-textbox edit without a linked eval run is the #1 regression source.
- Log **template ID + version as attributes AND rendered content in sampled payloads** — template-only can't show the bad variable substitution; rendered-only can't tell you which version to fix.
- Redact PII in-process before export, never in the backend; retention: content days-to-weeks, metadata months-to-years; scrub known-sensitive tool params by allowlist.
- Canary traffic is exempt from sampling entirely — it's the population gating a deploy.

## Sharpenings the strong baseline lacks

- **Tool-error-handled vs narrated-past**: log whether the model acknowledged a failed tool result or confidently narrated past it (tool span status=error followed by a confident final answer with no retry). The narrate-past case is the dangerous one and is detectable from spans alone — make it a first-class attribute, not a judge finding.
- **Trace-ID propagation is the usual silent gap**: retrieval service, agent loop, and the frontend feedback widget each minting their own IDs means the trace joins nowhere. W3C trace context end-to-end; the feedback widget carries the trace ID from response headers.
- **Judging only signal-flagged traces and reporting that pass-rate as "quality"** reports the failure population's pass-rate. The unbiased random judge sample is the headline metric; triggered judging is triage only. Annotate dashboards with judge-version boundaries so a rubric change reads as a step, not a regression.
- **Retrieval-miss framing**: if the answer wasn't in the retrieved context, the model didn't hallucinate — it was set up; and distinguish ingestion gap (doc not in index) from ranking failure — different fixes. Check retrieved doc IDs before model theories.
- **Clarification-spiral detection needs no judge**: 3+ ask-answer-ask cycles detectable from turn-role patterns alone; per-message quality can look fine while sessions fail.
- **Cost-per-hour runaway pages, monthly budgets don't**: a misconfigured retry/agent loop 100×'s spend before a budget alert fires; alert hourly token spend >3× same-hour baseline and agent budget-cutoff rate >1% (it burns money in real time).
- Quality SLOs phrased like availability SLOs ("≥97% of sampled traces pass groundedness, 7-day window") with an error budget that gates risky prompt experiments; every alert links its triggering traces.

## Verification / self-check — adequate when each takes ~5 minutes with tooling alone

1. "Show the exact rendered prompt/context for this complaint" (trace replay).
2. "Did quality change after yesterday's prompt merge?" (versioned deploy markers on judge/feedback dashboards).
3. "Top 3 failure clusters this week with example traces" (weekly trace mining: cluster negative-signal traces, read 5–10 per cluster, promote 2–3 to eval cases *before* fixing — the eval case proves the fix).
4. "Which feature spent the most tokens yesterday, and was it useful spend?" (attribution + outcome join; cost-per-successful-outcome is the north star).
5. "Would we notice within an hour if refusals tripled or an agent looped?" (fire a synthetic incident).

Stopping rule: instrument until those five work, then *consume* — the weekly mining ritual beats any additional dashboard. Eval-set growth of zero for a month means the loop is broken regardless of dashboard quality.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 delta (1 post-cutoff fact retained).
- Biggest baseline gaps found: OTel GenAI maturity detail (Development status, opt-in content capture, dual-emission flag) and the Langfuse/ClickHouse acquisition (Jan 2026); everything conceptual — tail sampling, judge hygiene, repair-rate leading indicator, loop meta-metric — was produced cold.
- Retained value: pinned 2026 ecosystem facts, span-level narrate-past detection, trace-ID propagation gap, and the flagged-traces-pass-rate trap.
