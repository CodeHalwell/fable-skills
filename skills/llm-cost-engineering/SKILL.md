---
name: llm-cost-engineering
description: Token economics and cost optimization for LLM applications — pricing mechanics (input/output asymmetry, caching discounts, batch APIs across Anthropic/OpenAI/Google), the optimization ladder in ROI order, prompt-cache engineering, cheap-model-first routing, output control, RAG-vs-long-context arithmetic, fine-tune break-even math, and cost monitoring with kill-switches. Load when analyzing, forecasting, or reducing LLM API spend, or designing cost-aware architecture.
---

# LLM Cost Engineering

## Core mental model

1. **Cost = (input tokens × input price) + (output tokens × output price × ~5-6) — and the multiplier is the headline.** As of mid-2026 every major provider prices output at roughly 5-6× input (Claude Opus 4.8: $5/$25 per MTok; GPT-5.5: $5/$30; Gemini 3.1 Pro: $2/$12). Consequence: a 500-token answer costs like 2,500-3,000 input tokens. "Make the model say less" is usually the cheapest optimization in the whole system, and it also cuts latency — output tokens dominate wall-clock time. Reasoning/thinking tokens bill as *output*, so a reasoning model at high effort can out-cost a nominally bigger model at low effort.
2. **Most input tokens in production are repeated tokens.** System prompt, tool definitions, few-shot examples, conversation history — resent on every request. Providers price repeated-prefix tokens at ~10% of base (Anthropic cache reads ~0.1×; OpenAI cached input $0.50 vs $5.00 on GPT-5.5; Gemini cached ~0.1× on 2.5+ models). An application that isn't achieving high cache-hit rates on stable prefixes is paying 10× for the same bytes. Cache-hit rate is a first-class production metric, like p99 latency.
3. **Prices are per token, but decisions are per task.** A cheaper-per-token model that needs 2 retries, writes 2× the tokens, or requires a 3× longer prompt can cost more per completed task. Different tokenizers produce different counts for the same text (Sonnet 5's tokenizer emits ~30% more tokens than Sonnet 4.6's for identical input). Always compare **$/successful task**, measured, never $/MTok read off a pricing page.
4. **Marginal cost per request is set by architecture, not by negotiation.** The levers, roughly in ROI order: shorter prompts and outputs → caching → routing cheap-model-first → batch API for async work → fine-tune-to-shrink. Each rung costs more engineering effort than the last; climb only as far as spend justifies.
5. **Unit economics or nothing.** Track cost per *task*, per *user*, per *feature* — not per request or per API key. An agent task is N requests with history resent each turn, so per-task cost grows superlinearly with turn count; per-request metrics hide this completely.
6. **Prices move; pin your assumptions.** Provider prices, cache TTLs, and discounts changed multiple times in 2024-2026 (e.g. Sonnet 5 launched with intro pricing $2/$10 through 2026-08-31, then $3/$15). Every cost model, dashboard, and break-even calc should embed the price table *with a date* and alert when actuals diverge.

## The cost model mechanics (verify prices before quoting — these are as of mid-2026)

**Representative per-MTok prices (as of mid-2026):**

| Tier | Anthropic | OpenAI | Google |
|---|---|---|---|
| Frontier | Fable 5: $10 / $50 | GPT-5.5: $5 / $30 | Gemini 3.1 Pro: $2 / $12 (≤200k ctx) |
| Workhorse | Opus 4.8: $5 / $25 · Sonnet 5: $3 / $15 | GPT-5.4: $2.50 / $15 | Gemini 3.5 Flash: $1.50 / $9 |
| Cheap | Haiku 4.5: $1 / $5 | GPT-5.4-mini: $0.75 / $4.50 · nano: $0.20 / $1.25 | Gemini 2.5 Flash: $0.30 / $2.50 |

Spread top-to-bottom is ~25-50×. That spread, not any single price, is why routing exists.

**Prompt caching semantics differ per provider — the differences change your design:**

- **Anthropic** — *explicit breakpoints*: `cache_control: {type: "ephemeral"}` on content blocks (max 4 per request), prefix-match on exact bytes in render order `tools → system → messages`. Reads ~0.1× base input. **Writes carry a surcharge**: 1.25× for the default 5-minute TTL, 2× for the 1-hour TTL. Minimum cacheable prefix is model-dependent (e.g. 4,096 tokens on Opus 4.8, 2,048 on Fable 5 / Sonnet 4.6) — shorter prefixes silently don't cache, no error. Verify via `usage.cache_read_input_tokens`.
- **OpenAI** — *automatic*: no code changes; any prefix ≥1,024 tokens matching a recent request is served cached (matched in 128-token increments). No write surcharge; cached-input price is listed per model (~90% off as of mid-2026, e.g. $0.50 vs $5.00 on GPT-5.5). Cache evicts after ~5-10 minutes of inactivity, ≤1 hour max. Verify via `usage.prompt_tokens_details.cached_tokens`.
- **Google Gemini** — *both*: implicit caching on by default for 2.5+ models (minimum ~2,048 tokens on 2.5, ~4,096 on 3.x; cached tokens ~0.1× — e.g. $0.20 vs $2.00 on 3.1 Pro), plus *explicit* CachedContent objects with a TTL where you additionally pay **storage per token-hour** (as of mid-2026 ~$4.50/MTok/hr on 3.1 Pro, ~$1/MTok/hr on Flash). Explicit caching is a paid reservation: worth it only when reuse within the TTL is guaranteed.

The common core: **caching is a prefix match on exact bytes**. All three reward the same layout discipline (below). The differences: Anthropic makes you place markers and charges for writes; OpenAI is free but you control nothing; Gemini lets you pay rent for guaranteed retention.

**Batch APIs:** all three providers offer ~50% off both input and output for asynchronous processing (Anthropic Message Batches: most complete <1h, 24h max, up to 100k requests/256MB; OpenAI Batch: 24h window; Gemini Batch: 50%). Discounts *stack* with caching (both Anthropic and OpenAI publish batch+cached rates). Anything that doesn't need an answer in seconds — nightly enrichment, evals, backfills, digest generation — belongs in batch. It's the highest discount per unit of engineering effort in the entire list: usually a day of work for a permanent 50%.

## The optimization ladder (ROI order, with the reasoning)

Climb in this order. Each rung: what it saves, what it costs you, when to stop.

1. **Prompt slimming + output control.** Savings: often 30-70%; effort: hours. Delete dead instructions, redundant few-shots, boilerplate ("You are a helpful..." paragraphs the model ignores), verbose JSON keys. Cap `max_tokens` per call-site at ~2× expected output; use stop sequences; demand structured output (a schema'd JSON answer is 5-10× shorter than prose restating the input, and it eliminates parse-failure retries — retry waste is a cost line most teams never see). Set thinking/effort budgets deliberately: on Anthropic, `output_config.effort` low/medium for routine work; reasoning tokens are output-priced. First because it's pure deletion — no infra, no quality risk if you eval the slimmed prompt, and it compounds with every later rung (a smaller prompt is cheaper to cache, cheaper to batch, cheaper on every model you route to).
2. **Prompt caching.** Savings: up to ~90% of *repeated-prefix* input; effort: hours to days (mostly reordering the prompt). Second because it's nearly free money on any workload with a stable system prompt or multi-turn history — but it requires the layout discipline below, and it only touches input cost.
3. **Model routing.** Savings: the 25-50× tier spread applied to the fraction of traffic that's easy; effort: weeks — you need a router *and an eval* to know the cheap model is good enough. Third because it's the first rung with real quality risk: never route without a regression eval gating it.
4. **Batch API.** Savings: flat 50% on eligible traffic; effort: days (queue + polling + result handling). Ranked after routing only because it applies solely to latency-insensitive traffic; on a workload that's mostly async, do it first — it's simpler than routing.
5. **Fine-tune-to-shrink.** Savings: replace a large-model + long-prompt combo with a small model that needs almost no prompt; effort: weeks-months (data curation, training, eval, *ongoing maintenance* — every base-model deprecation forces a re-train). Last because the training compute is cheap but the lifecycle isn't, and rungs 1-4 usually claw back 80%+ of waste first. Justified only by sustained volume on a narrow, stable task (break-even math in the worked examples).

Stopping rule: compute the ceiling before climbing. If remaining monthly spend on a workload is $500, rung 3+ can never repay its engineering cost — stop at caching. If spend is $50k/mo, every rung is on the table.

## Decision frameworks

**Cache-layout design — the questions in order:**
1. *What's byte-stable across requests?* Tools, system prompt, few-shots → front of the prompt, before any breakpoint. *What varies per session?* → next. *Per request* (user input, retrieved docs, timestamps) → last, after the final breakpoint.
2. *Is there a single moving element early?* One `datetime.now()` interpolated into the system prompt invalidates 100% of the cache 100% of the time — the prefix diverges at that byte and everything after it re-bills at full price. Audit: diff the rendered prompt bytes of two consecutive requests; the first divergent byte is where your cache ends.
3. *Which TTL?* (Anthropic) 5-min write = 1.25×, so break-even is 2 requests within 5 minutes; 1-h write = 2×, break-even ≥3 reads. Continuous traffic → 5-min (each request re-warms it). Bursty with gaps of 5-60 min → 1-h. Requests rarer than hourly → don't cache; you pay the surcharge and never read.
4. *What would change my mind?* Cache-read tokens near zero in `usage` despite "stable" prompts → a silent invalidator (unsorted JSON serialization, per-user tool sets, feature-flagged prompt sections) — fix the determinism, don't add more markers.

**Model routing — the reasoning chain:**
1. Segment traffic by task, not by request: which task types are narrow and verifiable (classify, extract, tag, reformat)? Those are cheap-model candidates; volume × tier-spread = the prize.
2. Build the eval *first*. Sample real traffic, label with the big model + human spot-check, then measure the cheap model's pass rate. The quality floor is a number ("≥97% agreement on the regression set"), not a vibe. No eval → no routing; a silent 5% quality regression can cost more in churn than the tokens saved.
3. Route cheap-first with escalation-on-failure: validation error, refusal shape, or low confidence → escalate one tier and retry once. This captures most of always-use-the-big-model quality at a fraction of cost, and it degrades gracefully — the failure signal is observable, unlike "the cheap model answered plausibly but wrong," which is why rung 2's eval must include exactly those plausible-but-wrong cases.
4. Re-run the router eval on every model version change and every prompt change. Routers rot.

**Batch or interactive — the questions in order:**
1. *Does anyone wait on this result?* A human staring at a spinner → interactive. A cron job, eval run, backfill, digest, enrichment pipeline → batch candidate.
2. *Is a 1-24h completion window acceptable to the downstream consumer?* Most Anthropic batches finish under an hour, but design for the 24h worst case: the consumer must tolerate results arriving any time in the window, in any order (key by `custom_id`, never position).
3. *Does it share a big prefix?* Put a `cache_control` breakpoint on the shared prefix inside the batch — batch 50% and cache ~90%-of-prefix discounts stack, so a shared-prefix batch job can run at ~5-10% of naive interactive cost.
4. *What would change my mind?* If the "async" job's results actually gate a user-facing flow within minutes, it's interactive traffic in disguise — batch will surface as mysterious product latency.

**Output-control levers, cheapest first:** (1) `max_tokens` cap per call-site (~2× expected output — a ceiling, not a target; the model doesn't see it); (2) stop sequences for delimited output; (3) structured output / JSON schema — shorter than prose *and* kills parse-retry waste; (4) explicit brevity instructions with positive examples ("respond in ≤3 sentences" beats "be concise"); (5) reasoning-token budgets — effort/thinking settings are output-priced cost levers, so route routine work at low effort and reserve high effort for tasks whose failure cost exceeds the token cost. Verbosity is a per-model property: re-measure output length whenever you change models, because a chattier model silently re-inflates the 5-6×-priced side of the bill.

**RAG vs. long-context — do the per-query arithmetic with caching factored in.** Corpus of 150k tokens, 500 queries/day against it, Opus 4.8 ($5/MTok in): stuffed uncached = 150k × $5/1M = **$0.75/query** ($375/day). Stuffed *cached* (continuous traffic, reads at ~0.1×) = **~$0.075/query** ($38/day). RAG at top-k ≈ 4k tokens = **$0.02/query** ($10/day) plus embedding/vector-store infra and a retrieval failure mode RAG adds. Pre-caching intuition ("long context is 37× RAG — must build RAG") is obsolete: cached long-context is within ~4× of RAG with zero retrieval risk. Decision: corpus fits in context *and* queries cluster within the cache TTL → stuff-and-cache; corpus far exceeds context, or is per-user (no shared prefix to cache), or queries are sparse → RAG. Re-run this arithmetic with your actual numbers; the crossover moves with corpus size, QPS, and provider.

## How an expert thinks through this

Scenario: a support-automation product spends $38k/month; the CFO wants it halved. The team proposes "switch everything to a cheaper model."

*First move: attribution before action.* Pull a week of token logs grouped by feature. Finding: 60% of spend is the agent-assist chat (interactive), 25% a nightly ticket-summarization job, 15% misc. Within chat, input tokens are 85% of cost — dominated by a 22k-token prefix (system prompt + tools + KB excerpt) resent every turn. **Reject the team's proposal as the first move**: model-swapping the chat feature risks quality on the flagship surface and doesn't touch the actual structure of the spend; the eval to de-risk it takes weeks. Park it as rung 3.

*Second: the nightly job is async and running on the interactive path.* Move it to the Batch API: flat 50% off 25% of spend, ~1 day of work, zero quality risk. Rejected alternative: routing the nightly job to a cheaper model *instead* — probably also fine, but batch is provable-safe today and the two stack; do batch now, evaluate the model swap later.

*Third: why is the 22k prefix billing at full price?* Check `usage.cache_read_input_tokens`: near zero. Diff two rendered prompts — byte 214 diverges: `Current time: 2026-07-05T14:03:22Z` interpolated into the system prompt. Move the timestamp to the end of the last user turn, add a `cache_control` breakpoint after the KB excerpt, verify reads go nonzero. That's ~85% × 60% of spend cut by ~80-90% for a two-line change. Rejected alternative: "shrink the 22k prompt itself first" — worth doing, but the cache fix is two lines with instant payoff; slimming is a follow-up that needs an eval run per deletion.

*Fourth: output control on chat.* Responses average 900 tokens; agents say customers skim. Add length guidance + `max_tokens: 700`. Output is 15% of chat cost but 5× priced — worth the hour.

*Stop.* Projected: batch (-12.5% total), cache fix (-40%), output (-4%) ≈ 55% reduction — target met without touching model choice. The router and fine-tune rungs stay parked until the *new* baseline justifies them. What would reopen them: volume 5×, or a new high-volume narrow task (e.g. auto-tagging) that a mini-tier model can pass at ≥97% on an eval.

## Failure modes & pitfalls

- **A timestamp, UUID, or user name early in the prompt zeroing the cache.** The classic. Prefix caching matches exact bytes; `f"Today is {date.today()}"` in the system prompt means no request ever hits cache. Symptom: `cache_read_input_tokens` / `cached_tokens` ≈ 0 with a "stable" prompt. Fix: dynamic content after the last breakpoint / at prompt end. Same bug wearing other clothes: `json.dumps` without `sort_keys=True`, dict/set iteration order in tool lists, per-user tool subsets, A/B-flagged prompt sections (every flag combo is a distinct prefix).
- **Cache markers on a prompt below the minimum cacheable size (Anthropic).** A 3k-token prefix on Opus 4.8 (minimum 4,096) silently never caches — no error, `cache_creation_input_tokens: 0`. Check the per-model minimum before concluding caching "doesn't work."
- **Paying the write surcharge with no reads.** Anthropic writes cost 1.25×/2×. Marking a prompt that's unique per request (fully dynamic RAG context, one-off jobs) *loses* money: 1.25× spent, 0.1× never collected. Cache only content with expected reuse ≥2 within TTL. Corollary: defaulting to the 1-h TTL "to be safe" doubles write cost and needs ≥3 reads to beat uncached — choose TTL from measured inter-arrival times.
- **Parallel fan-out paying N cold writes.** A cache entry becomes readable only once the first response begins streaming. Firing 50 concurrent requests with the same 20k prefix bills 50 full-price prefills. Fix: send 1, await first token, then release the other 49.
- **Editing the system prompt (or tool list, or model) mid-session.** Tools render at position 0; any change invalidates the entire conversation cache and the next request re-bills the whole history. Append mode-changes as messages instead of mutating the prefix; keep the tool list frozen per session.
- **Routing without an eval, or with a stale one.** "Haiku-tier handles this fine" judged by eyeballing 10 outputs → a plausible-but-wrong regression ships silently. The router needs a regression set with pass-rate gates, re-run on every prompt/model change. And measure *end-to-end*: if the cheap model fails validation 20% of the time and escalates, its effective cost includes those double-calls — sometimes the mid-tier model is cheaper per successful task.
- **Comparing models on input price alone.** Output is 5-6× input, verbosity varies by model, tokenizers differ (~±30% for the same text), and reasoning tokens bill as output. A "cheap" reasoning model at high effort on chatty defaults can exceed a frontier model at low effort with tight `max_tokens`. Only $/successful-task on your own traffic settles it.
- **No `max_tokens` discipline.** One "explain exhaustively" pathological output costs 50-100× median, and it recurs. Every call-site gets a cap ~2× expected output. Treat `stop_reason: max_tokens` on structured output as a validation failure (the JSON is broken), not as content — otherwise you pay again downstream.
- **Retry loops that resend the whole conversation.** A transient failure at turn 12 of an agent retried from turn 1 re-bills 12 turns of input. Retry the failed request only; make side effects idempotent so a retry is *possible* at request granularity. Related: appending the full error text of each failed attempt to the context — three retries later the prompt has tripled and the loop is more likely to fail, a cost *and* quality death spiral. Truncate/summarize error feedback.
- **Agent history growing quadratically.** Resending full history each turn means total input over a task ≈ turns² × avg-turn. A 40-turn agent session is not 2× a 20-turn one — it's ~4×. Levers: caching (mandatory — history is the ideal cached prefix), context editing/compaction of old tool results, and a turn cap. If per-task cost scales worse than linearly in your logs and cache reads are low, this is why.
- **Cost-per-request dashboards hiding cost-per-task explosions.** $0.004/request looks great while the agent quietly went from 8 to 30 requests per task after a prompt change. Instrument task IDs; alert on tokens-per-task and turns-per-task, not just spend-per-day.
- **No kill-switch on autonomous loops.** A looping agent (tool error → retry → same error) burns budget at machine speed all weekend. Every agentic loop needs a hard budget with an enforced stop (sketch below). Provider-side task budgets (e.g. Anthropic's `task_budget`) help the model self-moderate but are advisory — keep the hard stop client-side.
- **Fine-tuning for the wrong reason.** Fine-tuning to *inject knowledge* mostly fails (facts belong in RAG/context); fine-tuning to *shrink* — internalize format, style, task instructions, few-shots so a mini-tier model performs like a prompted large one on a narrow task — is the cost play. Also price the lifecycle: fine-tuned inference is often ~2× base-model rates (OpenAI, as of mid-2026), the model is pinned to a base snapshot so deprecations force re-training, and every product change to the task means new data + re-train + re-eval. Volume and task stability justify it; nothing else does.
- **Batch jobs on the interactive quota, or interactive traffic in batch.** Nightly batch through the sync API exhausts rate limits and 429s your users — and forfeits the 50% discount. Conversely, user-facing traffic pushed into batch to save money shows up as 24h-latency support tickets. Separate the paths; the batch API also isolates quota.
- **Trusting intro/temporary pricing in long-lived cost models.** Break-even calcs done at Sonnet 5's $2/$10 intro price are 50% wrong after 2026-08-31. Date-stamp every price in the model; alert when billed effective $/MTok diverges from the assumed table (this also catches provider price changes you missed).
- **Estimating tokens with the wrong tokenizer.** `len(text)//4` drifts badly on code, CJK, and numbers; `tiktoken` undercounts Claude tokens by ~15-20%+. Budget decisions ("will this corpus fit? what does this prompt cost?") use the provider's counting endpoint (`count_tokens`) against the *exact model* — counts differ even between adjacent model versions of the same family.
- **Long-context pricing tiers ignored.** Some models step-change price above a context threshold (Gemini 3.1 Pro: $2→$4/MTok input above 200k as of mid-2026). A retrieval setting that nudges average context from 190k to 210k doubles input cost with no visible code change. Alert on context-length distribution, not just token totals.
- **Explicit cache storage rent outliving its usefulness (Gemini).** An explicit CachedContent with a long TTL bills storage per token-hour whether or not anyone reads it. A 500k-token cache at ~$4.50/MTok/hr is ~$54/day of pure rent — forgotten test caches and over-long TTLs are recurring silent spend. Set TTLs from measured reuse windows and delete caches when jobs finish.
- **Provider-limit surprises breaking the caching plan.** Anthropic allows max 4 `cache_control` breakpoints and looks back only ~20 content blocks for a prior cache entry — an agent turn that adds 30 tool_use/tool_result blocks silently misses the previous turn's cache. Place an intermediate breakpoint inside long turns.

## Worked micro-examples (prices as of mid-2026 — re-verify before reusing)

**1. Caching a 20k-token system prompt — Claude Opus 4.8 ($5/MTok in, $25/MTok out), 10,000 requests/day, continuous traffic.** Each request: 20k stable prefix + 1k dynamic input + 500 output.

```text
Uncached:  input 21,000 × $5/1M = $0.1050/req   output 500 × $25/1M = $0.0125/req
           daily = 10,000 × $0.1175                        = $1,175/day
Cached:    prefix read 20,000 × $0.50/1M = $0.0100  (0.1×)
           dynamic     1,000 × $5/1M     = $0.0050
           output                        = $0.0125
           daily ≈ 10,000 × $0.0275 + writes             ≈ $275/day
Writes:    5-min TTL, continuous traffic keeps it warm; assume ~20 cold
           writes/day: 20 × 20,000 × $6.25/1M ≈ $2.50/day — noise.
Savings:   ~$900/day ≈ $27k/month ≈ 77% of total (86% of input cost),
           for reordering the prompt and one cache_control breakpoint.
```
Same shape on OpenAI (automatic): GPT-5.5 cached input $0.50 vs $5.00 → identical 90% prefix discount, zero code beyond keeping the prefix byte-stable.

**2. Fine-tune break-even — replace GPT-5.4 + fat prompt with fine-tuned GPT-4.1-mini on a narrow extraction task, 50,000 requests/day.**

```text
Option A  GPT-5.4 ($2.50/$15), 2,500-tok prompt (instructions+few-shots+input), 300 out:
          2,500×$2.5/1M + 300×$15/1M = $0.00625 + $0.0045 = $0.01075/req
          → $537/day ≈ $16.1k/month
Option B  fine-tuned GPT-4.1-mini (≈$0.80 in / $3.20 out as of mid-2026),
          prompt shrinks to 400 tok (task internalized by the fine-tune), 300 out:
          400×$0.80/1M + 300×$3.20/1M = $0.00032 + $0.00096 = $0.00128/req
          → $64/day ≈ $1.9k/month          savings ≈ $473/day
One-off:  training 5,000 examples × ~700 tok × 3 epochs ≈ 10.5M tok × ~$0.80/1M ≈ $8
          — the compute is trivial. Real cost: data curation + eval harness +
          integration ≈ 2 engineer-weeks ≈ $10-15k, plus re-train risk on base-
          model deprecation.
Break-even ≈ $12.5k / $473/day ≈ 26 days at this volume.
```
Same arithmetic at 2,000 requests/day: savings $19/day, break-even ~2 years → **don't**. The volume threshold, not the model quality, is the decision. Rule of thumb: fine-tune-to-shrink starts paying when (prompted cost − fine-tuned cost) × daily volume clears the loaded engineering + maintenance cost inside ~1 quarter.

**3. Runaway-loop kill-switch (per-task hard budget, client-side):**

```python
class TaskBudget:
    def __init__(self, usd_limit: float, prices: dict):   # prices date-stamped!
        self.spent, self.limit, self.p = 0.0, usd_limit, prices
    def record(self, usage):                              # call after EVERY response
        self.spent += (usage.input_tokens * self.p["in"]
                       + usage.output_tokens * self.p["out"]
                       + getattr(usage, "cache_read_input_tokens", 0) * self.p["cache_read"]
                       + getattr(usage, "cache_creation_input_tokens", 0) * self.p["cache_write"]) / 1e6
        if self.spent >= self.limit:
            raise BudgetExceeded(self.spent)              # hard stop the loop

budget = TaskBudget(usd_limit=2.00, prices=PRICES_2026_07)
while not done:
    resp = client.messages.create(...)
    budget.record(resp.usage)        # BudgetExceeded → halt, persist state, alert
    ...
```
Load-bearing details: count **all four** usage fields (cache writes cost more than base input); set the limit per *task class* (a $2 cap on a $0.05-median task = 40× headroom, catches loops without clipping legitimate work); on trip, persist partial state and page a human — don't silently retry, the loop *is* the bug. Add a second, slower fuse at the feature level (spend/hour vs trailing average) to catch fleets of individually-under-budget runaways.

## Verification / self-check

Before presenting any cost analysis or optimization plan:

- **Are all prices verified this session** (provider pricing pages / current docs), date-stamped, and marked "as of"? Never quote a cached-in-your-head price or TTL — these moved repeatedly through 2024-2026.
- **Does the arithmetic use the four-way token split** (uncached input, cache write, cache read, output) rather than a single blended input price? Recompute one example by hand: total prompt = `input_tokens + cache_creation + cache_read`.
- **Is the comparison per successful task**, including retries, escalations, and reasoning tokens — not per request or per MTok?
- **Cache claims validated against `usage`?** A caching recommendation is unverified until a real request shows nonzero cache reads; a routing recommendation is unverified without an eval pass-rate.
- **Did you check the quality side of every cost lever?** Any change touching model, prompt content, or output length names the eval that gates it. A cost win with an unmeasured quality regression is a loss.
- **Stopping rule:** projected savings of the next rung < its engineering cost (loaded, incl. maintenance) → stop and say so. Optimization below the noise floor of the bill is waste; so is a fine-tune pitch on a workload spending $300/month.
