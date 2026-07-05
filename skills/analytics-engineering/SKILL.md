---
name: analytics-engineering
description: Load when building or reviewing the analytics/warehouse layer — dbt projects, dimensional models, marts, metrics/semantic layers, warehouse cost or platform choices (Snowflake/BigQuery/Databricks/DuckDB), analytical SQL correctness (window functions, fan-out joins, null handling), or BI-layer architecture.
---

# Analytics Engineering

## Core mental model

1. **Grain declaration is step zero.** Every table has exactly one grain — "one row per X" — and
   every design conversation starts by stating it and writing it into the model's docs. Nearly
   every wrong number in analytics traces to a grain violation: a join that changed the grain, an
   aggregate computed at the wrong grain, a table whose grain silently drifted. If you can't state
   a table's grain in one sentence, the table is a bug in waiting.
2. **Dimensional modeling still wins.** Facts (events/measurements at a declared grain — mostly
   additive numerics plus foreign keys) and conformed dimensions (the nouns: customer, product,
   date) survived every "modern" alternative because they match how humans ask questions and how
   columnar engines execute. Wide denormalized marts are fine as a *final serving layer* — built
   *from* facts and dimensions, not instead of them.
3. **Transformations form a DAG of contracts, not a pile of queries.** Sources are declared,
   models reference each other via `ref()` (never hard-coded table names — that's what makes
   lineage, environments, and safe builds possible), tests are contracts enforced on every run,
   and layers have distinct jobs.
4. **Metrics must be defined once.** If "active users" is computed in four dashboards, there are
   four numbers. Push metric logic down — into marts and, where adopted, a semantic layer. The BI
   tool renders; it does not define.
5. **Warehouse cost is a design input, not a bill you discover.** Per-scan versus per-compute-time
   pricing changes which optimizations matter. Model with the meter in mind.
6. **Correctness beats cleverness in analytical SQL.** The classic analytics bugs — fan-out joins
   inflating sums, `NOT IN` with NULLs returning nothing, non-deterministic dedupe — are all
   *silent*. Plausible-but-wrong numbers are the failure mode; guard with tests, not vigilance.

## Layered dbt-era structure

State of the tooling (as of 2026): dbt remains the standard; the engine is mid-transition —
dbt Fusion (Rust, native SQL comprehension) rolling out, dbt Core v2 (Rust-based, Apache 2.0) in
alpha as of mid-2026, dbt Core 1.x (Python) still the widely deployed baseline; dbt Labs and
Fivetran announced a merger in 2026. SQLMesh is the credible alternative (column-level lineage,
virtual environments). The practices below are engine-independent.

- **staging** (`stg_`): 1:1 with source tables; rename, cast, light cleaning only. No joins, no
  business logic. Materialize as views. The only layer allowed to reference `source()`.
- **intermediate** (`int_`): reusable joins/derivations that don't deserve public exposure.
- **marts** (`fct_`, `dim_`): the products. Declared grain, tested keys, documented columns.
  Materialized as tables or incremental models.
- **Tests as contracts** — the minimum bar on every mart:
  - `unique` + `not_null` on the primary key. The PK uniqueness test is your automated fan-out
    detector: most upstream grain violations surface as PK duplicates. Teams that skip PK tests
    ship fan-out bugs, full stop.
  - `relationships` on foreign keys; `accepted_values` on enums; source freshness checks.
- **Incremental models and their late-data gotcha:**
  `WHERE event_ts > (SELECT max(event_ts) FROM {{ this }})` misses late-arriving rows forever.
  Use a lookback window (`event_ts > max - interval '3 days'`) with
  `incremental_strategy='delete+insert'` / `insert_overwrite` / `merge` plus a `unique_key`, and
  keep `--full-refresh` as the tested escape hatch. Reach for incremental only when the full
  rebuild is actually expensive — the default should be full rebuilds, because they're unbreakable.

## Dimensional modeling decisions

Reasoning chain for a new mart:

1. **What's the atomic business event?** That's the fact grain. Take the *lowest* grain you can
   afford (order line, not order): you can always aggregate up, never disaggregate down.
2. **Which measures are additive?**
   - Fully additive (amounts): sum across anything.
   - Semi-additive (balances, inventory): never sum across time — take last value or average.
   - Non-additive (ratios): store numerator and denominator; compute the ratio at query time.
     Averaging pre-computed ratios is a bug.
3. **Which dimension attributes need history?** The SCD decision, made per *attribute*, driven by
   one question: "when analyzing old facts, do users need this attribute *as it was then*?"
   - No → Type 1 (overwrite). Simplest; the default.
   - Yes → Type 2 (versioned rows with `valid_from`/`valid_to`/`is_current` and a surrogate key;
     facts join on the surrogate key valid at event time). dbt snapshots automate the capture.
   - Prior: teams over-apply Type 2 "to be safe" and drown in join complexity. Most attributes
     are honestly Type 1; reserve Type 2 for the few that answer real as-of questions (sales
     territory, subscription tier, customer segment).
4. **Conform the dimensions:** one shared `dim_customer` across all facts, or cross-fact analysis
   dies in reconciliation meetings.

## Warehouse platform and cost reasoning (as of 2026)

- **Snowflake / Databricks / BigQuery capacity mode** — you pay for *compute time*. Levers:
  warehouse/cluster size and runtime; aggressive auto-suspend; right-sizing per workload. The #1
  Snowflake waste is an idle-but-running warehouse.
- **BigQuery on-demand** — you pay per *bytes scanned*. Levers: partition pruning, clustering,
  never `SELECT *`. A dashboard refreshing an unpartitioned scan hourly is a money fire no matter
  how fast it runs.
- **Pruning levers per platform:** BigQuery partition + clustering keys; Snowflake clustering keys
  (only for multi-TB tables with consistent filter columns — clustering costs credits to maintain,
  don't cargo-cult it); Databricks liquid clustering; Iceberg/Delta partitioning + file sorting.
  Same principle everywhere: co-locate rows by the columns people filter on — usually date plus
  one entity.
- **DuckDB / MotherDuck** — the credible small warehouse. For sub-100GB-class analytics, dbt
  against DuckDB is faster and radically cheaper than a cloud warehouse, and DuckDB is the
  standard local dev/test target even when prod is Snowflake.
- **Freshness vs cost is a stated tradeoff:** every model gets an owner-approved freshness SLA
  (hourly? daily?) and is scheduled to *that* — not to "as often as possible". Sub-hourly
  freshness demands route to the streaming/CDC conversation, not to a 5-minute cron that rebuilds
  everything.

## Semantic layer judgment (as of 2026)

Landscape: dbt Semantic Layer (MetricFlow — open-sourced under Apache 2.0 in late 2025; the hosted
Semantic Layer API still requires the dbt platform), Cube (adds caching plus REST/SQL/GraphQL/MCP
serving), LookML as the legacy incumbent. The new adoption driver is AI: semantic layers as the
governance guardrail for LLM/agent-generated queries, exposed over MCP servers.

Judgment call:
- A semantic layer pays off when *many tools and consumers* need the *same metrics* — multiple BI
  tools, embedded analytics, AI agents.
- With one BI tool and a competent marts layer, defining metrics in well-tested marts plus the BI
  tool's modest modeling layer is less machinery for the same governance.
- Adopt when you catch the second definition of the same metric diverging — not before.
- Either way the invariant stands: **metric logic lives in exactly one governed place.**

## SQL craft: the bugs that produce wrong numbers

- **The fan-out join is THE classic analytics bug.** Joining orders (1 row/order) to order_items
  (N rows/order) and then `SUM(orders.amount)` counts each order's amount N times. The result
  looks plausible and is wrong. Discipline:
  - before writing any join, state both grains;
  - if the join is 1:N and you aggregate a 1-side measure, pre-aggregate the N side to the 1
    side's grain in a CTE first (see micro-example), or de-duplicate explicitly (last resort);
  - detect with `count(*)` before/after the join — unexpected growth = fan-out — plus PK
    uniqueness tests downstream.
- **NULL semantics in aggregates:**
  - `COUNT(col)` skips NULLs; `COUNT(*)` doesn't. `COUNT(converted_at) / COUNT(*)` is a correct
    conversion rate; `AVG(CASE WHEN converted THEN 1 END)` without `ELSE 0` computes a different
    denominator — sometimes intended, usually a bug.
  - `NOT IN (subquery)` returns zero rows if the subquery yields any NULL — use `NOT EXISTS`.
  - `WHERE x != 'foo'` silently drops NULL rows.
  - `SUM` over zero rows is NULL, not 0 — `COALESCE(SUM(x), 0)` at serving boundaries.
- **Non-deterministic dedupe:** `ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at)` with
  tied timestamps picks an arbitrary row per run. Add a unique tiebreaker to the ORDER BY. Any
  "latest record" logic without a total ordering flaps between runs and gaslights everyone
  comparing dashboards.
- **Window function craft:**
  - Know the default frame: `AVG(x) OVER (ORDER BY d)` defaults to `RANGE ... CURRENT ROW` — a
    running average, not the whole partition.
  - Rolling windows need explicit `ROWS BETWEEN 27 PRECEDING AND CURRENT ROW` — and that's 28
    *rows*, which equals 28 *days* only when there's exactly one row per day. Gaps break it;
    densify against a date spine first.
  - `RANK` vs `DENSE_RANK` vs `ROW_NUMBER` differ exactly when ties exist — pick deliberately.
- **Timezone drift:** decide once (store UTC, convert at the edge). `DATE(ts)` in the warehouse's
  default timezone vs the business timezone moves revenue across day boundaries and creates
  unreconcilable off-by-one-day reports.

## Self-serve, managed honestly — and BI-layer anti-patterns

The self-serve dream fails in two directions: "everyone queries raw tables" (chaos, wrong numbers)
and "every question needs an engineer" (bottleneck). The honest middle:
- a small set of governed, documented, tested marts/metrics covering ~80% of questions;
- a clearly-labeled exploration space (sandbox schema, ad-hoc SQL) whose outputs are understood
  to be unaudited;
- a paved road for promoting a sandbox insight into a governed model.

BI-layer anti-patterns to reject in review:
- Business logic in dashboard-level custom SQL or calculated fields — invisible to lineage and
  tests; move it into a model.
- Dashboard-to-dashboard copy-paste of metric definitions.
- Page filters that quietly change a metric's meaning relative to its name.
- Extract/import pipelines inside the BI tool creating a second, stale warehouse.

## How an expert thinks through it: "dashboard revenue doesn't match finance"

*Two numbers disagree — first establish which is wrong against the source of truth (the billing
system export). Don't debug the pipeline yet; debug the definitions.* Finance matches billing;
the dashboard is 4% high.

*Prior for inflated: fan-out join. Prior for deflated: filters or joins dropping rows. Inflated,
so hunt the fan-out.* Read the mart's lineage: `fct_revenue` joins `stg_invoices` to
`stg_invoice_line_items` to attach product category — and sums `invoice.total` *after* the join.
There it is: multi-line invoices counted once per line.

Considered and rejected:
- `SUM(DISTINCT total)` — rejected: breaks the moment two invoices legitimately share an amount.
- Patching the dashboard with a correction factor — rejected with prejudice: logic in the BI layer,
  and the mart stays broken for every other consumer.

Fix: restate the grain. `fct_revenue` becomes one row per *invoice line* with `line_amount`
(fully additive); invoice-level totals live in their own model at their own grain. Add a `unique`
test on the line PK and a reconciliation test — `SUM(line_amount)` vs the billing-system control
total, failing at >0.5% divergence.

*A 0.3% gap remains after the fix.* Timezone: the dashboard buckets by UTC date, finance by
fiscal-local date. Align the definition inside the metric and document it. Close by checking
lineage for other consumers of the broken mart and notifying them with corrected historicals —
silent fixes to money metrics destroy trust faster than the bug did.

## Worked micro-example: pre-aggregate to kill a fan-out

```sql
-- Grain contract: fct_orders = one row per order_id.
with line_items as (            -- collapse the N side to the 1 side's grain FIRST
    select
        order_id,
        sum(quantity * unit_price)        as items_amount,
        count(*)                          as line_count,
        coalesce(sum(discount_amount), 0) as discount_amount
    from {{ ref('stg_order_items') }}
    group by 1
)
select
    o.order_id,
    o.customer_id,
    o.ordered_at,
    li.items_amount,
    li.line_count,
    li.discount_amount
from {{ ref('stg_orders') }} o
left join line_items li using (order_id);
-- Tests: unique(order_id), not_null(order_id), plus a singular test asserting
-- sum(items_amount) matches the staging total within tolerance.
```

## Verification / self-check

- State the grain of every table you created or touched, one sentence each. Verify it by actually
  running `SELECT pk, count(*) ... GROUP BY 1 HAVING count(*) > 1` (or the `unique` test).
- For every join you wrote: what are the two grains, and did the row count change as expected
  across it?
- Reconcile at least one headline number against an independent source — the source system's
  total, or the previously accepted number with the diff explained.
- Sweep for the NULL traps in anything you wrote: `NOT IN`, `!=` filters, `COUNT(col)` vs
  `COUNT(*)`, un-defaulted CASE inside aggregates, implicit window frames.
- Cost pass: does the new model prune partitions and avoid `SELECT *`; is its schedule matched to
  its freshness SLA rather than "hourly by default"?
- Stopping rule: a mart is done when its grain, tests, docs, and one reconciliation check exist
  and pass — not when it answers every hypothetical future question. Resist speculative columns
  and premature incrementalization; add them when a real query needs them.
