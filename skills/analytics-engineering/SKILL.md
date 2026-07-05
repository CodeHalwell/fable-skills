---
name: analytics-engineering
description: Load when building or reviewing the analytics/warehouse layer — dbt projects, dimensional models, marts, metrics/semantic layers, warehouse cost or platform choices (Snowflake/BigQuery/Databricks/DuckDB), analytical SQL correctness (window functions, fan-out joins, null handling), or BI-layer architecture.
---

# Analytics Engineering

Frontier models already know grain discipline, fan-out joins, NULL traps, window frames, SCD choice, and warehouse cost levers cold. This sheet keeps the checklists, the 2026 tooling anchors, and the few sharpenings a cold model lacks.

## Anchors

1. **Grain declaration is step zero** — "one row per X," written into docs, enforced by a `unique` test on the grain key. Nearly every wrong number is a grain violation; the PK uniqueness test is the automated fan-out detector.
2. Facts + conformed dimensions still win; wide denormalized marts are a *serving layer built from* them, not a replacement.
3. `ref()` everywhere, layers with distinct jobs: `stg_` (1:1 source, rename/cast only, views, sole `source()` caller) → `int_` → marts (`fct_`/`dim_`, declared grain, tested keys). Minimum mart bar: `unique`+`not_null` on PK, `relationships` on FKs, `accepted_values` on enums, source freshness.
4. Metrics defined once — in marts or a semantic layer; the BI tool renders, it does not define.
5. Fan-out fix: pre-aggregate the N side to the 1 side's grain in a CTE *before* joining (never `SUM(DISTINCT)` — breaks on legitimately-equal amounts). Detect: `count(*)` before/after the join.
6. Incremental models: `WHERE event_ts > (SELECT max(event_ts) FROM {{ this }})` misses late rows forever → lookback window + `delete+insert`/`insert_overwrite`/`merge` with `unique_key`, `--full-refresh` tested. Default to full rebuilds — they're unbreakable; incrementalize only when rebuilds are actually expensive.

## Tooling anchors (verified 2026 — cold models are one release behind)

- dbt engine is mid-transition: **dbt Fusion** (Rust, native SQL comprehension) rolling out; **dbt Core v2 — Rust-based and Apache 2.0 — in alpha as of mid-2026** (the reflex "Fusion is source-available, Core stays Python" is stale); dbt Core 1.x still the deployed baseline. **dbt Labs and Fivetran announced a merger in 2026.** SQLMesh remains the credible alternative (column-level lineage, virtual environments).
- Semantic layer: **MetricFlow open-sourced under Apache 2.0 in late 2025** (the hosted Semantic Layer API still requires the dbt platform); Cube adds caching + REST/SQL/GraphQL/**MCP** serving. The 2026 adoption driver is AI governance — semantic layers as the guardrail for LLM/agent-generated queries, exposed over MCP servers. Adopt when you catch the second diverging definition of the same metric, or when many tools/agents consume the same metrics — not before; one BI tool + tested marts needs less machinery.
- DuckDB/MotherDuck: the credible small warehouse; sub-100GB analytics on dbt+DuckDB is faster and radically cheaper, and DuckDB is the standard local dev/test target even when prod is Snowflake.

## SQL traps (review checklist — one line each)

- Fan-out join then `SUM` of the 1-side measure (state both grains before every join).
- `COUNT(col)` vs `COUNT(*)`; `AVG(CASE WHEN ... THEN 1 END)` without `ELSE 0` changes the denominator.
- `NOT IN (subquery)` with a NULL → zero rows (use `NOT EXISTS`); `WHERE x != 'foo'` drops NULLs.
- `SUM` over zero rows is NULL → `COALESCE` at serving boundaries.
- Dedupe without a unique tiebreaker in ORDER BY flaps between runs (and check DESC for "latest").
- Default window frame is `RANGE ... CURRENT ROW` (running, and tie-inclusive); `ROWS BETWEEN 27 PRECEDING` = 28 *rows*, equal to 28 *days* only on a densified date spine.
- Timezone: store UTC, convert at the edge; `DATE(ts)` in warehouse-default vs business timezone moves revenue across day boundaries.

## Cost levers (compressed)

Per-compute-time (Snowflake/Databricks/BQ capacity): idle warehouses and right-sizing; aggressive auto-suspend. Per-bytes-scanned (BQ on-demand): partition pruning, clustering, never `SELECT *`. Snowflake clustering keys only for multi-TB tables with consistent filter columns — reclustering burns credits; don't cargo-cult. Every model gets an owner-approved freshness SLA and is scheduled to *that*; sub-hourly demands route to the CDC/streaming conversation, not a 5-minute cron.

## SCD in one rule

Per *attribute*, one question: "analyzing old facts, do users need this attribute as it was then?" No → Type 1 (the honest default for most attributes). Yes → Type 2 (`valid_from/valid_to/is_current`, surrogate key, facts join the version valid at event time; dbt snapshots capture). Teams over-apply Type 2 "to be safe" and drown in join complexity.

## The debugging walkthrough distilled: "dashboard ≠ finance"

Establish which number is wrong against the source of truth *before* debugging the pipeline. Inflated → hunt fan-out (found: invoice total summed after joining line items). Restate the grain (one row per invoice line, additive `line_amount`), don't patch (`SUM(DISTINCT)` and BI-layer correction factors rejected with prejudice). Add the reconciliation test vs the billing control total (fail >0.5%). Residual 0.3% → timezone bucketing. Close by notifying every consumer of the broken mart with corrected historicals — silent fixes to money metrics destroy trust faster than the bug did.

## Self-serve + BI anti-patterns (reject on sight)

Governed marts covering ~80% of questions + labeled sandbox + a paved promotion road. Reject: business logic in dashboard custom SQL/calculated fields; dashboard-to-dashboard metric copy-paste; page filters silently changing a metric's meaning; BI-tool extract pipelines forming a second stale warehouse.

## Worked micro-example: pre-aggregate to kill a fan-out

```sql
-- Grain contract: fct_orders = one row per order_id.
with line_items as (            -- collapse the N side to the 1 side's grain FIRST
    select order_id,
           sum(quantity * unit_price)        as items_amount,
           count(*)                          as line_count,
           coalesce(sum(discount_amount), 0) as discount_amount
    from {{ ref('stg_order_items') }}
    group by 1
)
select o.order_id, o.customer_id, o.ordered_at,
       li.items_amount, li.line_count, li.discount_amount
from {{ ref('stg_orders') }} o
left join line_items li using (order_id);
-- Tests: unique(order_id), not_null(order_id), + singular reconciliation vs staging total.
```

## Verification / self-check

- State every touched table's grain in one sentence; verify with the `unique` test, not by assertion.
- For every join: both grains stated, row-count change explained.
- Reconcile one headline number against an independent source.
- Sweep the SQL-trap checklist above on anything you wrote.
- Cost pass: pruning, no `SELECT *`, schedule matched to freshness SLA.
- Done = grain + tests + docs + one reconciliation passing; resist speculative columns and premature incrementalization.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 delta.
- Opus cold nails: grain/fan-out discipline, the full NULL-trap list, RANGE-default + 28-rows-vs-days, SCD per-attribute rule, incremental lookback pattern, Snowflake clustering caveat, non-deterministic dedupe.
- Sharpened: dbt Core v2 (Rust, Apache 2.0, alpha mid-2026) vs the stale "Fusion is the only Rust engine" picture; MetricFlow Apache-2.0 relicense + MCP/AI-governance adoption driver; Fivetran merger dated 2026.
