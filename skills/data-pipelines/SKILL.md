---
name: data-pipelines
description: Load when designing, reviewing, or debugging batch data pipelines — ETL/ELT jobs, orchestration (Airflow/Dagster/Prefect), incremental loads, backfills, partitioned lake/warehouse storage, or when deciding whether Spark/streaming is needed at all. Covers idempotency, watermarks, failure recovery, and pipeline testing.
---

# Batch Data Pipeline Engineering

## Core mental model

1. **Idempotency is the prime directive.** Every pipeline run WILL be re-run: by a retry, a backfill,
   an on-call engineer at 3am, or a scheduler bug that double-fires. Design every task so running it
   twice with the same inputs produces the same end state. Anything else is a latent data-corruption
   bug with a fuse of unknown length.
2. **A pipeline is a function of (logical date, input snapshot) → output partition.** Not "whatever
   data exists right now" → "append somewhere". If you can't say what a run for `2026-07-04` would
   write without knowing wall-clock time, the pipeline isn't deterministic and can't be safely re-run.
3. **Backfill is a first-class operation, not an emergency procedure.** The question "how do I
   recompute the last 90 days?" must have a boring answer on day one. Pipelines that can't backfill
   get frozen: nobody dares fix logic bugs because history can't be repaired.
4. **Transform where the compute is (ELT over ETL).** Move raw data to the warehouse/lake cheaply and
   dumbly; transform in the engine that has the data, elastic compute, and SQL. Custom transform code
   running on the extraction box is where logic goes to become unmaintainable.
5. **The failure you must design for is partial failure.** Whole-run failure is easy: rerun it. The
   killer is 40 of 50 partitions written, then a crash. Idempotent overwrite-by-partition turns
   partial failure into "rerun, done"; append turns it into a forensic investigation.
6. **Alert on data SLAs, not task status.** A green DAG that produced an empty table is a worse
   failure than a red DAG, because nobody looks at it. "Did the data arrive, is it fresh, is the
   volume in the expected band" are the real health checks.

## Idempotency in practice

The canonical pattern is **overwrite-partition** (delete-insert / `INSERT OVERWRITE`):

- Each run owns a partition keyed by the *logical* date/hour — the scheduler-supplied interval,
  never `datetime.now()`.
- The run computes the full contents of that partition and atomically replaces it:
  - `INSERT OVERWRITE PARTITION` on engines that support it;
  - `DELETE WHERE ds = X` + `INSERT` inside one transaction;
  - `CREATE OR REPLACE TABLE ... AS SELECT` for full-snapshot tables;
  - `MERGE` on a natural key when the target isn't partition-aligned.
- Re-run ⇒ same partition replaced ⇒ same state. Double-fire ⇒ harmless. Backfill ⇒ just a loop
  over logical dates.

**The append-only trap.** `INSERT INTO target SELECT ...` looks innocent and passes every test,
because tests run once. The first retry duplicates rows; the duplicates inflate every downstream
aggregate; the inflation is discovered weeks later by a confused analyst. When you see append in a
scheduled job, ask "what happens on retry?" — if the answer is "duplicates", it's a bug even if it
has never fired. Appends are acceptable only for:
- immutable event logs *with* a downstream dedupe key, or
- loads guarded by a run-ID / staging-table-swap protocol that makes the commit atomic.

Non-obvious idempotency leaks to check in review:
- `now()` / `CURRENT_TIMESTAMP` in business logic (use the logical date).
- Random UUIDs generated during the run and used as join keys (change on rerun).
- Reading "all files currently in the landing bucket" instead of the run's explicit manifest.
- Side effects (emails, API posts, cache invalidations) fired before the write commits.
- Auto-increment surrogate keys that renumber on rerun and break downstream joins.

## Incremental processing design

Reasoning chain for "full refresh or incremental?":

1. **How big is the full recompute?** If full refresh runs in minutes and costs cents — very common
   under ~100GB on a modern warehouse or DuckDB — *use full refresh*. Incremental logic is a
   complexity loan; don't take it out to save $0.40/day. This is the decision teams most often get
   wrong-way-round: they pay complexity to optimize a job that was already cheap.
2. If incremental is warranted, **what is the watermark?** Prefer an immutable, monotonic column:
   a source-maintained `updated_at`, a CDC log sequence number, a file's landing timestamp. Store
   the high-watermark from the *previous successful run* in state; never hard-code "yesterday".
3. **How late does data arrive?** Events dated Tuesday can land Thursday (mobile clients, upstream
   retries, timezone bugs). Pick a reprocess window: each run recomputes the last N days of
   partitions (`WHERE ds >= logical_date - INTERVAL '3 days'`, overwriting those partitions).
   N comes from *measuring* actual lateness, plus margin — not from a guess.
4. **Keep the escape hatch.** Every incremental model needs a documented, tested full-refresh path
   (dbt: `--full-refresh`; custom: a backfill entrypoint parameterized by date range). Incremental
   state *will* drift or corrupt eventually; the fix must be one command, not archaeology.

`updated_at`-based incrementals silently miss two things — ask which the source does before trusting:
- **Hard deletes**: the row vanishes; no watermark ever sees it. You need CDC or a periodic full
  reconciliation pass.
- **Updates when you filtered on `created_at`**: mutated history never re-enters the window.

## Orchestration (landscape as of 2026)

- **Airflow 3.x** (GA 2025): asset-aware scheduling, DAG versioning, multi-team deployments; the
  largest ecosystem and hiring pool. Default for teams that already run it or have a platform team.
- **Dagster**: asset/lineage-centric with quality checks first-class; the best dbt integration and
  developer experience. Winning greenfield modern-data-stack teams.
- **Prefect**: lightest, Python-function-first, modest ops overhead; added its own asset layer.
  Good when orchestration needs are simple and the team is small.
- All three converged on asset-based scheduling; the task-vs-asset mental model now matters more
  than the brand. Don't relitigate the choice mid-project without a concrete failure it fixes.

Principles that outlast the tool:
- The DAG is a **dependency contract**: downstream must not run on upstream failure or emptiness.
  Data-interval alignment matters — a daily job consuming an hourly job depends on all 24 hourly
  partitions, not on "the hourly DAG ran recently".
- Tasks exchange **references** (table names, partition keys, paths) — never dataframes through the
  orchestrator's metadata channel (XCom-style). The orchestrator schedules; the engine computes.
- **Backfill through the orchestrator**, so one code path parameterized by logical date serves both
  daily runs and history rebuilds. A separate backfill script always diverges from the DAG.
- Set catchup/backfill policy explicitly. Airflow catchup on a new DAG with an old `start_date`
  firing hundreds of surprise runs is a classic self-inflicted incident.

## Right-sizing compute: the "you don't need Spark" arithmetic

Before reaching for distributed anything, do the arithmetic:

- A single node with 64–256GB RAM running **DuckDB** (1.4 LTS as of 2026; reads/writes Parquet and
  Iceberg) or **Polars** (lazy, streaming engine) handles working sets into the hundreds of GB.
  DuckDB scans ~100GB of Parquet on a laptop in well under a minute; both spill to disk.
- The threshold question is not "is my data big?" but "is my *per-job working set after partition
  pruning and column projection* bigger than one big machine?" A 10TB table processed one daily
  partition at a time is a 30GB problem.
- Choose Spark / warehouse-scale engines when the working set genuinely exceeds a node, when you
  need cluster concurrency, or when the org already standardizes there — not because "pipeline"
  reflexively means "cluster". The Spark tax (cluster ops, serialization, executor debugging, cold
  starts) is paid on every run forever.

## Storage: files, partitioning, formats (as of 2026)

- **Partition by query pattern, not by write convenience.** Consumers filter by event date and maybe
  tenant — partition by those, even if the producer finds it easier to dump by arrival hour.
  Partitioning by arrival time when everyone queries event time forces full scans forever.
- **Cardinality rule:** partitions are for pruning, not uniqueness. Thousands of partitions: fine.
  Millions (partitioning by `user_id`): metadata explosion plus the small-files problem.
  High-cardinality access belongs in file-level sort order / clustering, not the partition scheme.
- **Small files kill throughput** (per-file open/list/footer overhead). Target ~100MB–1GB Parquet
  files; run compaction as scheduled maintenance. Frequent small ingests into a lake without
  compaction always degenerate — plan compaction from day one.
- **Formats:** Parquet is the universal file format. For *tables* on object storage use an open
  table format:
  - **Apache Iceberg** is the 2026 default for new open lakehouses — its REST catalog is the de
    facto standard; AWS S3 Tables, Snowflake, and BigQuery all offer managed Iceberg.
  - **Delta Lake** remains the default inside Databricks with the largest installed base; interop
    layers (UniForm, Apache XTable) are converging the formats.
  - Table formats buy atomic overwrite, schema evolution, time travel, hidden partitioning — they
    make the idempotency patterns above cheap. Raw Parquet-directories-as-tables is legacy practice
    for anything with concurrent writers.

## Failure design

- **Poison records:** one malformed row must not kill a million-row batch. Parse defensively at the
  boundary; route failures to a **dead-letter table** carrying raw payload, error, and run ID.
  Alert on dead-letter *rate*, not existence. Never silently drop — silent drops are how
  "revenue is 2% low" mysteries are born.
- **Retries with idempotency, or not at all.** Retrying a non-idempotent task converts a transient
  failure into corruption. Fix idempotency before adding `retries=3`.
- **Data SLAs per critical table:** freshness (max staleness), volume (row count within a band
  around the trailing average), key quality (PK uniqueness, non-null FKs). Enforce two ways:
  - as pipeline steps that *fail the run* (dbt tests, assertion tasks), and
  - as an independent freshness monitor that pages when the table is stale *regardless of why* —
    this catches the scheduler-never-fired and green-but-empty cases task alerts miss.
- **Recovery drill:** the runbook for "yesterday's data is wrong" should read: fix code → rerun
  affected logical dates → downstream recomputes via its own idempotent runs. If any step is
  "manually delete rows", the design is incomplete.

## Testing pipelines

- **Fixture data over mocks:** a small checked-in dataset with the edge cases (nulls, late rows,
  duplicates, unicode, one poison record) fed through the *real* transform locally. DuckDB is
  superb for this — it runs most warehouse-flavored SQL against the same Parquet.
- **Test invariants, not snapshots:** row-count preservation across joins (fan-out check), PK
  uniqueness, sums preserved through reshaping. Golden-file snapshot tests break on every
  intentional change and get deleted within a quarter.
- **The idempotency test:** literally execute the task twice in the test and assert identical
  end state. This one test catches the whole append-trap class.
- **Dev/prod parity:** the classic failure is dev pointing at tiny synthetic data, so performance
  and data-quality bugs appear only in prod. Mitigations:
  - a staging environment fed a deterministic sample of production (`hash(user_id) % 100 = 0`);
  - shadow runs — run the new version against staging for a few cycles and diff outputs against
    prod before cutover.

## How an expert thinks through it: "daily orders pipeline shows inflated revenue"

*Inflated, not missing — so extra rows, not lost rows. Prior: duplicates from a retry against an
append-only load. Check scheduler history first, before reading any SQL — cheaper evidence.*

Airflow shows the load task failed and retried on July 2. *Does the load append?* Yes:
`INSERT INTO fact_orders SELECT ...`. *But one INSERT...SELECT is atomic — a failed first attempt
shouldn't have committed anything. So how did the retry duplicate?* Look closer: the task first
copies files to a staging prefix, then loads "all files in staging". The retry re-copied files that
were already there under new names. The non-idempotent step is the file copy, not the SQL.

Considered and rejected along the way:
- *"The warehouse partially committed the INSERT"* — rejected: engines almost never partially commit
  a single statement; suspect your own glue code first.
- *Add `SELECT DISTINCT` downstream* — rejected: hides the defect, breaks legitimately-identical
  rows, and leaves the corrupt fact table in place.

Fix: convert the load to overwrite-by-partition keyed on logical date; staging prefix includes the
run ID and is wiped at task start; rerun July 2 to repair history. Then add the PK-uniqueness test
that would have caught this on day one, and reconcile July 2–4 totals against the source system to
confirm closure.

## Worked micro-example: idempotent incremental with a late-data window

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

The dbt equivalent: `materialized='incremental'` with `incremental_strategy='delete+insert'`
(or `insert_overwrite` on partition-aware platforms), the same 3-day lookback inside the
`is_incremental()` filter, and `--full-refresh` kept as the tested escape hatch.

## Verification / self-check

Before presenting a pipeline design or fix, confirm:
- Run it twice on the same logical date — is the end state identical? If you can't answer, you're
  not done.
- What exactly happens on: task retry mid-write; a 3-day-late event; a backfill of last quarter;
  an empty source day; a poison record? Each needs a one-sentence boring answer.
- Is anything keyed on wall-clock time or "files currently present"? Replace with logical dates
  and manifests.
- Would a green run with zero output rows page anyone? If not, add the volume/freshness SLA check.
- Did you pick the smallest sufficient engine (warehouse SQL / DuckDB before Spark), justifying
  anything bigger with working-set arithmetic?

Stopping rule: stop hardening when every failure mode above has a rehearsed recovery of "rerun the
affected dates". Don't add config-driven generality, exotic engines, or streaming to a daily batch
need — the next engineer maintains what you wrote.
