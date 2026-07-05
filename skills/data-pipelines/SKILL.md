---
name: data-pipelines
description: Load when designing, reviewing, or debugging batch data pipelines — ETL/ELT jobs, orchestration (Airflow/Dagster/Prefect), incremental loads, backfills, partitioned lake/warehouse storage, or when deciding whether Spark/streaming is needed at all. Covers idempotency, watermarks, failure recovery, and pipeline testing.
---

# Batch Data Pipeline Engineering

Frontier models already reason well about pipeline idempotency, watermarks, and orchestration. This sheet keeps only the anchors, checklists, and corrections that add precision — treat it as a review checklist, not a tutorial.

## Core rules (anchors, not explanations)

1. **Every run WILL be re-run.** A pipeline is a function of (logical date, input snapshot) → output partition. Overwrite-by-partition (`INSERT OVERWRITE` / delete+insert in one txn / `CREATE OR REPLACE` / `MERGE`); appends only for immutable logs with a downstream dedupe key or a run-ID staging-swap protocol.
2. **Backfill is a first-class operation** — one code path parameterized by logical date serves daily runs and history rebuilds; a separate backfill script always diverges.
3. **Alert on data SLAs, not task status.** Freshness = `max(event_time)` recency + volume band + key quality; an independent monitor pages on staleness *regardless of why* — catches green-but-empty and scheduler-never-fired, which task alerts structurally miss.
4. **Full refresh is the default.** Under ~100GB / minutes of runtime / cents per run on a modern warehouse or DuckDB, incremental logic is a complexity loan taken out to save $0.40/day — the decision teams most often get wrong-way-round.
5. **Reprocess window from measurement:** each incremental run recomputes the trailing N days of partitions, where N = *measured* p99 event lateness + margin, not a guess. `updated_at` watermarks silently miss hard deletes and updates outside the filter window — pair with CDC or periodic full reconciliation, and keep a tested `--full-refresh` escape hatch.

## Idempotency-leak review checklist (each is a one-question audit)

- `now()`/`CURRENT_TIMESTAMP` in business logic (use the logical date)
- Run-generated UUIDs used as join keys (change on rerun)
- Reading "all files currently in the bucket" instead of the run's manifest
- Side effects (emails, API posts, cache busts) fired before the write commits
- Auto-increment surrogate keys that renumber on rerun
- Watermark advanced before the write commits (crash = permanent gap)
- Staging prefix not keyed by run ID / not wiped at task start

## Landscape anchors (verified 2026)

- **Airflow 3.x** (GA 2025: asset-aware scheduling, DAG versioning, multi-team) — default for incumbents; **Dagster** — asset/lineage-first, best dbt integration, winning greenfield; **Prefect** — lightest. All three converged on asset-based scheduling; the task-vs-asset mental model now matters more than the brand. Watch Airflow catchup on a new DAG with an old `start_date` — the classic surprise-hundred-runs incident.
- **DuckDB 1.4 LTS** reads *and writes* Parquet and Iceberg; scans ~100GB of Parquet on a laptop in well under a minute; Polars streams larger-than-RAM. The threshold question is the *per-job working set after pruning and projection*, not table size — a 10TB table processed one daily partition at a time is a 30GB problem. The Spark tax (cluster ops, executor debugging, cold starts) is paid on every run forever.
- **Iceberg** is the 2026 default for new open lakehouses (REST catalog de facto standard; managed by S3 Tables, Snowflake, BigQuery); **Delta** remains the Databricks default; UniForm/XTable converge them. Raw Parquet-directories-as-tables is legacy for anything with concurrent writers.
- Partition by query pattern (event date + maybe tenant), never by arrival convenience. Thousands of partitions fine; millions (per-user) = metadata explosion. High-cardinality access goes in file sort order/clustering. Target ~100MB–1GB files; schedule compaction from day one.
- Orchestrator hygiene: tasks exchange references (table names, partition keys), never dataframes through XCom-class channels; a daily job consuming an hourly job depends on all 24 partitions, not "the hourly DAG ran recently."

## The debugging pivot worth memorizing

*Inflated numbers after a retry* → the reflex is "the warehouse partially committed the INSERT." Wrong: engines almost never partially commit a single statement. **Suspect your own glue first** — the classic culprit is a non-idempotent step *around* the SQL: a file copy into a staging prefix that re-copied under new names on retry, a manifest that picked up both attempts' files. Fix the load to overwrite-by-partition, key staging by run ID, rerun the affected dates — and reject `SELECT DISTINCT` downstream (hides the defect, breaks legitimately-identical rows, leaves the fact table corrupt).

## Testing

- **The idempotency test**: execute the task twice in the test, assert identical end state — one test kills the whole append-trap class.
- **PK uniqueness at the output grain + volume band** — the highest-value pair; catches fan-outs and duplicate loads.
- Fixture data (nulls, late rows, duplicates, one poison record) through the *real* transform; DuckDB runs most warehouse-flavored SQL locally. Invariant tests (row-count preservation, sum preservation) over golden-file snapshots — snapshots get deleted within a quarter.
- Poison records: dead-letter table with raw payload + error + run ID; alert on *rate*, never silently drop.
- Dev/prod parity: deterministic production sample (`hash(user_id) % 100 = 0`) and shadow runs diffed against prod before cutover.

## Worked micro-example: idempotent incremental with a late-data window

```sql
-- :ds = logical date. Recomputes a 3-day window (measured lateness) idempotently.
BEGIN;
DELETE FROM analytics.fact_orders
 WHERE order_date BETWEEN :ds - INTERVAL '2 days' AND :ds;
INSERT INTO analytics.fact_orders
SELECT o.order_id, o.order_date, o.customer_id, o.amount_usd
FROM raw.orders o
WHERE o.order_date BETWEEN :ds - INTERVAL '2 days' AND :ds;
COMMIT;
-- Rerunnable, backfillable (any :ds), late-tolerant, deterministic (no now(), no external state).
```

dbt equivalent: `incremental_strategy='delete+insert'` (or `insert_overwrite`), same lookback inside `is_incremental()`, `--full-refresh` kept tested.

## Verification / self-check

- Run twice on the same logical date — identical end state?
- One-sentence boring answers for: retry mid-write; 3-day-late event; backfill of last quarter; empty source day; poison record.
- Anything keyed on wall-clock or "files currently present"? Replace with logical dates and manifests.
- Would a green run with zero rows page anyone?
- Smallest sufficient engine, justified by working-set arithmetic?

Stopping rule: stop hardening when every failure mode has a rehearsed recovery of "rerun the affected dates." Don't add config-driven generality, exotic engines, or streaming to a daily batch need.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 11 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Opus cold nails idempotent overwrite, watermark blind spots, full-refresh-first economics, working-set-vs-Spark arithmetic, Iceberg-default landscape, Airflow3/Dagster/Prefect positioning, the idempotency-leak list, and idempotency/PK testing.
- Restructured to a correction sheet; retained value = the review checklists, verified version anchors (DuckDB 1.4 LTS Iceberg writes, Airflow 3.x specifics), and the "single statements don't partially commit — suspect your glue" debugging pivot.
