---
name: analytics-engineering
description: Load when building or reviewing the analytics/warehouse layer — dbt projects, dimensional models, marts, metrics/semantic layers, warehouse cost or platform choices (Snowflake/BigQuery/Databricks/DuckDB), analytical SQL correctness (window functions, fan-out joins, null handling), or BI-layer architecture.
---

# Analytics Engineering

## Core mental model

1. **Grain declaration is step zero.** Every table has exactly one grain — "one row per X" — and every design conversation starts by saying it out loud and writing it in the model's docs. Nearly every wrong number in analytics traces to a grain violation: a join that changed the grain, an aggregate computed at the wrong grain, a table whose grain silently drifted. If you can't state a table's grain in one sentence, the table is a bug in waiting.
2. **Dimensional modeling still wins.** Facts (events/measurements at a declared grain, mostly additive numerics + foreign keys) and conformed dimensions (the nouns: customer, product, date) survived every "modern" alternative because they match how humans ask questions and how columnar engines execute them. Wide denormalized marts are fine as a *final serving layer*, but they're built *from* facts and dimensions, not instead of them.
3. **Transformations form a DAG of contracts, not a pile of queries.** The dbt-era insight: sources are declared, models reference each other via `ref()` (never hard-coded table names — that's what makes lineage, environments, and safe builds possible), tests are contracts enforced on every run, and layers have distinct jobs.
4. **Metrics must be defined once.** If "active users" is computed in four dashboards, there are four numbers. Push metric logic down: into marts, and (where adopted) a semantic layer. The BI tool renders; it does not define.
5. **Warehouse cost is a design input, not a bill you discover.** Per-scan (BigQuery on-demand) vs per-compute-time (Snowflake warehouses, Databricks clusters) pricing changes which optimizations matter. Model with the meter in mind.
6. **Correctness beats cleverness in analytical SQL.** The classic analytics bugs — fan-out joins inflating sums, `NOT IN` with NULLs returning nothing, non-deterministic dedupe — are all *silent*. SQL that returns plausible-but-wrong numbers is the failure mode; guard with tests, not vigilance.

## Layered dbt-era structure (as of 2026)

State of the tooling: dbt remains the standard; the engine is mid-transition — dbt Fusion (Rust, native SQL comprehension) in rollout, dbt Core v2 (Rust-based, Apache 2.0) in alpha as of mid-2026, dbt Core 1.x (Python) still the widely deployed baseline; dbt Labs and Fivetran announced a merger in 2026. SQLMesh exists as the credible alternative (column-level lineage, virtual environments). The *practices* below are engine-independent.

- **staging** (`stg_`): 1:1 with source tables; rename, cast, light cleaning only. No joins, no business logic. Materialize as views. This is the only layer allowed to reference `source()`.
- **intermediate** (`int_`): reusable joins/derivations that don't deserve exposure. Keep private.
- **marts** (`fct_`, `dim_`): the products. Declared grain, tested keys, documented columns. Materialize as tables or incremental.
- **Tests as contracts:** minimum bar on every mart — `unique` + `not_null` on the primary key, `relationships` on foreign keys, `accepted_values` on enums, plus source freshness checks. A uniqueness test on the PK is your automated fan-out detector: most grain violations upstream surface as PK duplicates. Teams that skip PK tests ship fan-out bugs, full stop.
- **Incremental models and their late-data gotcha:** `WHERE event_ts > (SELECT max(event_ts) FROM {{ this }})` misses late-arriving rows forever. Use a lookback window (`event_ts > max - interval '3 days'`) with `incremental_strategy='delete+insert'`/`insert_overwrite`/`merge` and a `unique_key`, and keep `--full-refresh` as the tested escape hatch. Reach for incremental only when full rebuild is actually expensive — the default should be full table rebuilds because they're unbreakable.

## Dimensional modeling decisions

Reasoning chain for a new mart:
1. **What's the atomic business event?** That's your fact grain — take the *lowest* grain you can afford (order line, not order): you can always aggregate up, never disaggregate down.
2. **Which measures are additive?** Amounts sum across everything; balances/inventory are semi-additive (never sum across time — take last value or average); ratios are non-additive (store numerator and denominator, compute the ratio at query time — averaging ratios is a bug).
3. **Which dimensions need history?** This is the SCD decision, per *attribute*, driven by one question: "when analyzing old facts, do users need the attribute *as it was then*?" No → Type 1 (overwrite; simplest, default). Yes → Type 2 (versioned rows with `valid_from`/`valid_to`/`is_current` and a surrogate key; facts join on the surrogate key valid at event time). dbt snapshots automate Type 2 capture. Prior: teams over-apply Type 2 "to be safe" and drown in join complexity — most attributes are honestly Type 1; reserve Type 2 for the few that answer real "as-of" questions (sales territory, subscription tier, customer segment).
4. **Conform the dimensions:** one `dim_customer` shared by all facts, or cross-fact analysis dies.

## Warehouse platform and cost reasoning (as of 2026)

- **Snowflake / Databricks / BigQuery (capacity mode):** you pay for *compute time* → the levers are warehouse/cluster size and runtime; auto-suspend aggressively, right-size per workload, watch for idle-but-running warehouses (the #1 Snowflake waste).
- **BigQuery on-demand:** you pay per *bytes scanned* → the levers are partition pruning, clustering, and never `SELECT *`; a dashboard refreshing an unpartitioned-scan query hourly is a money fire regardless of how fast it runs.
- **Pruning levers per platform:** BigQuery partition + clustering keys; Snowflake clustering keys (only for multi-TB tables with consistent filter columns — clustering costs credits to maintain, don't cargo-cult it); Databricks liquid clustering; Iceberg/Delta partitioning + file sorting. Same principle everywhere: co-locate rows by the columns people filter on (usually date + one entity).
- **DuckDB / MotherDuck:** the credible small-warehouse option — for sub-100GB-class analytics, running dbt against DuckDB is faster and radically cheaper than a cloud warehouse, and it's the standard local dev/test target even when prod is Snowflake.
- **Freshness vs cost is a stated tradeoff:** every model gets an owner-approved freshness SLA (hourly? daily?) and is scheduled to *that*, not to "as often as possible". Sub-hourly freshness demands should route to the streaming/CDC discussion, not to a 5-minute dbt cron that rebuilds everything.

## Semantic layer judgment (as of 2026)

The landscape: dbt Semantic Layer (MetricFlow — open-sourced Apache 2.0 in late 2025; the hosted API still requires the dbt platform), Cube (adds caching + REST/SQL/GraphQL/MCP serving), Looker's LookML as the legacy incumbent, plus AI-agent access as the new driver (semantic layers as the guardrail for LLM-generated queries — MCP servers exposing governed metrics). Judgment: a semantic layer pays off when *many tools/consumers* ask for the *same metrics* — multiple BI tools, embedded analytics, AI agents. If you have one BI tool and a competent marts layer, defining metrics in well-tested marts + the BI tool's modest layer is less machinery for the same governance. Adopt the semantic layer when you catch the second definition of the same metric diverging, not before. Either way the rule stands: **metric logic lives in exactly one governed place.**

## SQL craft: the bugs that produce wrong numbers

- **The fan-out join is THE classic analytics bug.** Joining orders (1 row/order) to order_items (N rows/order) then `SUM(orders.amount)` counts each order's amount N times. The number looks plausible and is wrong. Discipline: before writing any join, state both grains; if the join is 1:N and you aggregate a left-side measure, you must either pre-aggregate the N side to the 1 side's grain in a CTE, or aggregate with `COUNT(DISTINCT order_id)`-style de-duplication (last resort). Detection: `SELECT count(*)` before and after the join — unexpected growth = fan-out; and PK uniqueness tests downstream.
- **NULL semantics in aggregates:** `COUNT(col)` skips NULLs, `COUNT(*)` doesn't — a "conversion rate" of `COUNT(converted_at)/COUNT(*)` is right, `AVG(CASE WHEN ... THEN 1 END)` without `ELSE 0` is a different (sometimes intended, usually not) denominator. `NOT IN (subquery)` returns zero rows if the subquery yields any NULL — use `NOT EXISTS`. `x != 'foo'` silently drops NULL rows. Sums of no rows are NULL, not 0 — `COALESCE(SUM(x),0)` at serving boundaries.
- **Non-deterministic dedupe:** `ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at)` with tied timestamps picks an arbitrary row per run — add a unique tiebreaker to the ORDER BY. Any "latest record" logic without a total ordering flaps between runs and gaslights everyone comparing dashboards.
- **Window function craft:** know the frame — `AVG(x) OVER (ORDER BY d)` defaults to `RANGE ... CURRENT ROW` (running average), not the whole partition; rolling windows need explicit `ROWS BETWEEN 27 PRECEDING AND CURRENT ROW` (and that's 28 *rows*, which equals 28 *days* only if the input has exactly one row per day — gaps break it; densify with a date spine first). `RANK` vs `DENSE_RANK` vs `ROW_NUMBER` differ exactly when ties exist.
- **Timezone drift:** decide once (store UTC, convert at the edge); `DATE(ts)` in the warehouse's default TZ vs the business TZ moves revenue across day boundaries and creates unreconcilable off-by-one-day reports.

## Self-serve and BI-layer discipline

The self-serve dream fails when it means "everyone queries raw tables" (chaos) or "every question needs an engineer" (bottleneck). The honest middle: a small set of governed, documented, tested marts/metrics that cover ~80% of questions, plus a clearly-labeled exploration space (sandbox schema, ad-hoc SQL) whose outputs are understood to be unaudited — and a paved road for promoting a sandbox insight into a governed model. BI-layer anti-patterns to reject in review: business logic in dashboard-level custom SQL or calculated fields (it's invisible to lineage and tests — move it to a model), dashboard-to-dashboard copy-paste definitions, filters that quietly change a metric's meaning per-page, and extract/import pipelines inside the BI tool that create a second, stale warehouse.

## How an expert thinks through it: "revenue on the exec dashboard doesn't match finance's number"

Internal monologue: *Two numbers disagree — first establish which is wrong, against source-of-truth (billing system export). Don't debug the pipeline yet; debug the definitions.* Finance matches billing; the dashboard is 4% high. *Prior for inflated: fan-out join. Prior for deflated: filters/joins dropping rows. Inflated, so hunt the fan-out.* Read the mart's lineage: `fct_revenue` joins `stg_invoices` to `stg_invoice_line_items` to bring in product category — and sums `invoice.total` post-join. There it is: multi-line invoices counted once per line. *Considered: `SUM(DISTINCT total)` — rejected, breaks when two invoices legitimately share an amount. Considered: patching the dashboard with a divisor — rejected with prejudice, logic in the BI layer.* Fix: restate the grain — `fct_revenue` should be one row per *invoice line* with `line_amount` (fully additive), while invoice-level totals live at their own grain; add `unique` test on the line PK and a reconciliation test: `SUM(line_amount)` vs billing-system control total, failing at >0.5% divergence. *Remaining 0.3% gap after the fix?* Timezone: dashboard buckets by UTC date, finance by fiscal-local date — align the definition in the metric, document it. Close by checking who else consumed the broken mart (lineage), and notify with the corrected historical numbers — silent fixes to money metrics destroy trust.

## Worked micro-example: pre-aggregate to kill a fan-out

```sql
-- Grain contract: fct_orders = one row per order_id.
with line_items as (            -- collapse N-side to the 1-side's grain FIRST
    select order_id,
           sum(quantity * unit_price)             as items_amount,
           count(*)                               as line_count,
           coalesce(sum(discount_amount), 0)      as discount_amount
    from {{ ref('stg_order_items') }}
    group by 1
)
select o.order_id, o.customer_id, o.ordered_at,
       li.items_amount, li.line_count, li.discount_amount
from {{ ref('stg_orders') }} o
left join line_items li using (order_id);
-- Tests: unique(order_id), not_null(order_id),
-- plus a singular test asserting sum(items_amount) matches stg totals within tolerance.
```

## Verification / self-check

- State the grain of every table you created or touched, in one sentence each. Verify with `SELECT pk, count(*) ... GROUP BY 1 HAVING count(*) > 1` (or the `unique` test) — actually run it.
- For every join you wrote: what are the two grains, and did row count change as expected across it?
- Reconcile at least one headline number against an independent source (source system total, or the previous accepted number with the diff explained).
- Check the NULL traps in anything you wrote: `NOT IN`, `!=` filters, `COUNT(col)` vs `COUNT(*)`, un-defaulted CASE in aggregates, window frames.
- Cost pass: does the new model prune partitions / avoid `SELECT *`; is its schedule matched to its freshness SLA?
- Stopping rule: a mart is done when its grain, tests, docs, and one reconciliation check exist and pass — not when it answers every hypothetical future question. Resist speculative columns and premature incrementalization; add them when a real query needs them.
