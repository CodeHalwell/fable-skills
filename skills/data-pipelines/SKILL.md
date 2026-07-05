---
name: data-pipelines
description: Load when designing, reviewing, or debugging batch data pipelines — ETL/ELT jobs, orchestration (Airflow/Dagster/Prefect), incremental loads, backfills, partitioned lake/warehouse storage, or when deciding whether Spark/streaming is needed at all. Covers idempotency, watermarks, failure recovery, and pipeline testing.
---

# Batch Data Pipeline Engineering

## Core mental model

1. **Idempotency is the prime directive.** Every pipeline run WILL be re-run: by a retry, a backfill, an on-call engineer at 3am, or a scheduler bug that double-fires. Design every task so running it twice with the same inputs produces the same end state. Anything else is a latent data-corruption bug with a fuse of unknown length.
2. **A pipeline is a function of (logical date, input snapshot) → output partition.** Not "whatever data exists right now" → "append somewhere". If you can't say what a run for `2026-07-04` would write without knowing wall-clock time, the pipeline isn't deterministic and can't be safely re-run.
3. **Backfill is a first-class operation, not an emergency procedure.** The question "how do I recompute the last 90 days?" must have a boring answer on day one. Pipelines that can't backfill get frozen: nobody dares fix logic bugs because history can't be repaired.
4. **Transform where the compute is (ELT over ETL).** Move raw data to the warehouse/lake cheaply and dumbly; do transformation in the engine that has the data, elastic compute, and SQL. Custom transform code running on the extraction box is where logic goes to become unmaintainable.
5. **The failure you must design for is partial failure.** Whole-run failure is easy (rerun it). The killer is: 40 of 50 partitions written, then a crash. Idempotent overwrite-by-partition turns partial failure into "rerun, done"; append turns it into a forensic investigation.
6. **Alert on data SLAs, not task status.** A green DAG that produced an empty table is a worse failure than a red DAG, because nobody looks at it. "Did the data arrive, is it fresh, does it have roughly the expected volume" are the real health checks.

## Idempotency in practice

The canonical pattern is **overwrite-partition** (a.k.a. delete-insert or `INSERT OVERWRITE`):

- Each run owns a partition keyed by the *logical* date/hour (the scheduler-supplied interval, never `datetime.now()`).
- The run computes the full contents of that partition and atomically replaces it: `INSERT OVERWRITE`, `DELETE WHERE ds = X` + `INSERT` in one transaction, `CREATE OR REPLACE TABLE ... AS` for full snapshots, or `MERGE` keyed on a natural key when the target isn't partition-aligned.
- Re-run ⇒ same partition replaced ⇒ same state. Double-fire ⇒ harmless.

**The append-only trap:** `INSERT INTO target SELECT ...` looks innocent and passes every test, because tests run once. The first retry duplicates rows; the duplicates inflate every downstream aggregate; the inflation is discovered weeks later by a confused analyst. If you see append in a scheduled job, ask "what happens on retry?" — if the answer is "duplicates", it's a bug even if it has never fired yet. Appends are acceptable only for immutable event logs *with* a downstream dedupe key, or when guarded by a run-ID/staging-swap protocol.

Non-obvious idempotency leaks to check: `now()` or `CURRENT_TIMESTAMP` in business logic (use logical date), random UUIDs as join keys, reading "all files in the landing bucket" instead of the run's manifest, side effects (emails, API posts) fired before the write commits, and auto-increment surrogate keys that change on rerun.

## Incremental processing design

Reasoning chain for "full refresh or incremental?":
1. **How big is the full recompute?** If the full-refresh runs in minutes and costs cents (very common under ~100GB on a modern warehouse or DuckDB), *use full refresh*. Incremental logic is a complexity loan; don't take it out to save $0.40/day. This is the most commonly wrong-way-round decision.
2. If incremental is warranted: **what is the watermark?** Prefer an immutable, monotonic column (`updated_at` maintained by the source, a CDC log sequence number, a file's landing time). Store the high-watermark from the *previous successful run* in state, not "yesterday" hard-coded.
3. **How late does data arrive?** Events dated Tuesday can land Thursday (mobile clients, upstream retries, timezone bugs). Pick a reprocess window: each run recomputes the last N days of partitions, not just today's (`WHERE ds >= logical_date - INTERVAL '3 days'`, overwriting those partitions). N comes from measuring actual lateness, plus margin.
4. **Keep the escape hatch.** Every incremental model needs a documented, tested full-refresh path (dbt: `--full-refresh`; custom: a backfill entrypoint parameterized by date range). Incremental state *will* drift or corrupt eventually; the fix must be one command.

`updated_at`-based incrementals silently miss: hard deletes (row vanishes, watermark never sees it — you need CDC or periodic full reconciliation), and in-place updates when you filtered on `created_at`. Ask which the source does before trusting the column.

## Orchestration (as of 2026)

Landscape: **Airflow 3.x** (GA 2025; asset-aware scheduling, DAG versioning, largest ecosystem — default for teams that already run it or have a platform team), **Dagster** (asset/lineage-centric, best dbt integration and dev experience — winning greenfield modern-data-stack teams), **Prefect** (lightest, Python-function-first — good when orchestration needs are modest). All three now do asset-based scheduling; the task-vs-asset mental model matters more than the brand.

Principles that outlast the tool:
- The DAG is a **dependency contract**: downstream must not run on upstream failure or emptiness. Data-interval alignment matters — a daily job consuming an hourly job must depend on all 24 hours, not "the hourly DAG ran recently".
- Tasks exchange **references** (table/partition names, paths), never dataframes through the orchestrator's metadata channel (XCom-style). The orchestrator schedules; the engine computes.
- **Backfill through the orchestrator**, so the same code path, parameterized by logical date, serves both daily runs and history rebuilds. If backfill is a separate script, the two will diverge.
- Set `catchup`/backfill policy explicitly. Airflow's catchup on a new DAG with an old `start_date` firing hundreds of runs is a classic self-inflicted incident.

## Right-sizing compute: the "you don't need Spark" arithmetic

Before reaching for Spark/distributed anything, do the arithmetic: a single node with 64–256GB RAM running **DuckDB** (1.4 LTS as of 2026, reads/writes Parquet and Iceberg) or **Polars** (streaming engine, lazy queries) handles working sets into the hundreds of GB — DuckDB scans ~100GB of Parquet on a laptop in under a minute, and both spill to disk. The threshold question is not "is my data big?" but "is my *per-job working set after partition pruning and column projection* bigger than one big machine?" A 10TB table processed one daily partition at a time is a 30GB problem. Choose Spark/BigQuery/Snowflake-scale engines when the working set genuinely exceeds a node, when you need the cluster's concurrency, or when the org already standardizes there — not because "pipeline" reflexively means "cluster". The tax of Spark (cluster ops, serialization, debugging executors, cold starts) is paid on every run forever.

## Storage: files, partitioning, formats (as of 2026)

- **Partition by query pattern, not by write convenience.** Consumers filter by event date and maybe tenant — so partition by those, even if the producer finds it easier to dump by arrival hour. Partitioning by arrival time when everyone queries event time forces full scans forever.
- **Cardinality rule:** partitions are for pruning, not uniqueness. Thousands of partitions: fine. Millions (e.g., partition by `user_id`): metadata explosion and the small-files problem. High-cardinality access goes in sort order/clustering *within* files, not in the partition scheme.
- **Small files kill throughput** (open/list/footer overhead per file). Target ~100MB–1GB Parquet files; compact as a scheduled maintenance job. Streaming ingestion into a lake without compaction always degenerates — plan compaction from day one.
- **Formats:** Parquet is the universal file format. For *tables* on object storage, use an open table format — **Apache Iceberg is the 2026 default for new open lakehouses** (REST catalog is the de facto standard; AWS S3 Tables, Snowflake, BigQuery all support managed Iceberg); **Delta Lake** remains the default inside Databricks with the largest installed base, and interop (UniForm, Apache XTable) is converging the two. Table formats buy you atomic overwrite, schema evolution, time travel, and hidden partitioning — i.e., they make the idempotency patterns above cheap. Raw Parquet-directories-as-tables is legacy practice for anything multi-writer.

## Failure design

- **Poison records:** one malformed row must not kill a million-row batch. Parse defensively at the boundary; route failures to a **dead-letter table** with the raw payload, error, and run ID; alert on dead-letter *rate*, not existence. Never silently drop — silent drops are how "revenue is 2% low" mysteries are born.
- **Retries with idempotency, or not at all.** Retrying a non-idempotent task converts a transient failure into corruption. Fix idempotency before adding `retries=3`.
- **Data SLAs:** for each critical table define freshness (max staleness), volume (row count within expected band vs trailing average), and key quality checks (uniqueness of PK, non-null FKs). Run them *as pipeline steps that fail the run* (dbt tests, or assertions), plus an independent freshness monitor that pages when the table is stale regardless of why. Task-level alerting misses the worst failures: the job that "succeeds" doing nothing.
- **Recovery drill:** the runbook for "yesterday's data is wrong" should be: fix code → rerun affected logical dates → downstream recomputes via its own idempotent runs. If any step is "manually delete rows", the design is incomplete.

## Testing pipelines

- **Fixture data over mocks:** a small, checked-in dataset with edge cases (nulls, late rows, duplicates, unicode, a poison record) fed through the *real* transform locally (DuckDB is superb for this — it runs most warehouse-flavored SQL and reads the same Parquet).
- **Test invariants, not snapshots:** row-count preservation across joins (fan-out check), PK uniqueness, sums preserved through reshaping. Golden-file snapshot tests break on every intentional change and get deleted.
- **The dev/prod parity problem:** the classic failure is dev pointing at tiny synthetic data, so performance and data-quality bugs only appear in prod. Mitigate with a staging environment fed by a sampled-but-real slice of production (deterministic sample, e.g., `hash(user_id) % 100 = 0`), and run new pipeline versions against staging for a few cycles comparing outputs to prod (shadow run) before cutover.
- Always test: rerun-twice-equals-run-once (idempotency test — literally execute the task twice in the test and assert identical state).

## How an expert thinks through it: "daily orders pipeline is showing inflated revenue"

Internal monologue: *Inflated, not missing — so extra rows, not lost rows. Prior: duplicates from a retry against an append-only load. Check the scheduler history first, before reading any SQL — cheaper evidence.* Airflow shows the load task failed and retried on July 2. *Does the load append?* Yes: `INSERT INTO fact_orders SELECT ...`. *The first attempt had partially committed before dying? No — it's one INSERT...SELECT, atomic. So how did a retry duplicate?* Look closer: the task first copies files to a staging prefix, then loads "all files in staging". The retry re-copied files that were already there under new names — the file copy is the non-idempotent step, not the SQL. *Considered and rejected:* blaming the warehouse ("transaction didn't roll back") — warehouses almost never partially commit a single statement; suspect your own glue code first. Also rejected: adding `SELECT DISTINCT` downstream — that hides the defect and breaks legitimately-identical rows. Fix: load becomes overwrite-by-partition keyed on logical date, staging prefix includes run ID and is wiped at task start, and a rerun of July 2 repairs history. Then add the PK-uniqueness test that would have caught this on day one, and rerun July 2–4 to verify the numbers reconcile against the source system.

## Worked micro-example: idempotent incremental with late-data window

```sql
-- Runs daily with :ds = logical date. Recomputes a 3-day window to absorb late events.
BEGIN;
DELETE FROM analytics.fact_orders
 WHERE order_date BETWEEN :ds - INTERVAL '2 days' AND :ds;
INSERT INTO analytics.fact_orders
SELECT o.order_id, o.order_date, o.customer_id, o.amount_usd
FROM raw.orders o
WHERE o.order_date BETWEEN :ds - INTERVAL '2 days' AND :ds;
COMMIT;
-- Properties: rerunnable (delete+insert in one txn), backfillable (any :ds),
-- late-tolerant (3-day window), deterministic (no now(), no state outside the table).
```

Equivalent in dbt: `materialized='incremental'`, `incremental_strategy='delete+insert'` (or `insert_overwrite` on partitioned platforms), with the same 3-day lookback in the `is_incremental()` filter — and remember `--full-refresh` remains the escape hatch.

## Verification / self-check

Before presenting a pipeline design or fix, confirm:
- Run it twice on the same logical date — is the end state byte-identical? If you can't answer, you're not done.
- What exactly happens on: task retry mid-write, a 3-day-late event, a backfill of last quarter, an empty source day, a poison record? Each needs a one-sentence boring answer.
- Is anything keyed on wall-clock time or "files currently present"? Replace with logical date and manifests.
- Would a green run with zero output rows page anyone? If not, add the volume/freshness SLA check.
- Did you pick the smallest sufficient engine (warehouse SQL/DuckDB before Spark) and justify anything bigger with working-set arithmetic?

Stopping rule: stop hardening when every failure mode above has a rehearsed recovery of "rerun the affected dates". Don't add config-driven generality, exotic engines, or streaming for a daily batch need — the next engineer maintains what you wrote.
