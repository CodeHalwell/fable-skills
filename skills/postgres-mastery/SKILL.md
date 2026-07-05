---
name: postgres-mastery
description: Load for PostgreSQL work beyond basic CRUD — query performance and EXPLAIN analysis, index selection, vacuum/bloat/autovacuum issues, connection pooling, lock contention or migration outages, replication choices, extension selection (pgvector/PostGIS/Timescale-class), or Postgres version/upgrade decisions.
---

# PostgreSQL Mastery

## Core mental model

1. **MVCC explains everything weird about Postgres.** UPDATE = insert a new row version + mark the
   old one dead; DELETE = mark dead. Dead tuples are reclaimed only by (auto)vacuum, and vacuum can
   only reclaim versions no *currently running transaction* could still see. From this one fact
   derive: table and index bloat, why long-running transactions are poison (they pin dead tuples
   database-wide — idle-in-transaction sessions included), why `count(*)` isn't free, why
   write-heavy tables need vacuum tuning, and the transaction-ID wraparound emergency.
2. **The planner is a cost model fed by statistics — argue with it via evidence.** It ignores your
   index for enumerable reasons: stale stats, low selectivity, type or expression mismatch, or a
   correct judgment that a seq scan is cheaper. `EXPLAIN (ANALYZE, BUFFERS)` is the conversation;
   guessing is not.
3. **Every index is a write tax and a vacuum burden.** Indexes cost slower writes, more WAL, and —
   when you index a frequently-updated column — loss of heap-only-tuple (HOT) update eligibility.
   Index what queries need, prove it with usage stats, drop the rest.
4. **Connections are expensive; pool them.** One backend process per connection, with memory and
   scheduling costs to match. Every serious deployment fronts Postgres with PgBouncer or an
   equivalent; the app-side "just raise max_connections" instinct makes things worse.
5. **DDL takes locks, and locks queue.** An "instant" `ALTER TABLE` blocked behind one long query
   blocks *everyone behind it* — the classic self-inflicted outage. Migrations are lock-scheduling
   problems first, schema problems second.
6. **The extension ecosystem is the moat.** pgvector, PostGIS, TimescaleDB-class time-series,
   pg_partman, pg_stat_statements. The expert's first question about "should we add specialized
   database X?" is "does a Postgres extension do this well enough to avoid a second system?"
   The answer is often yes.

Current baseline (as of 2026): **PostgreSQL 18** (released September 2025) is current. Headline
features: an asynchronous I/O subsystem (large gains on sequential reads and vacuum), native
`uuidv7()` (time-ordered UUIDs — prefer over v4 for B-tree-friendly keys), B-tree **skip scan**
(multicolumn indexes usable when the leading column isn't filtered — helps, but don't design
around it), virtual generated columns (now the default for generated columns), temporal PK/FK
constraints, `OLD`/`NEW` in `RETURNING`, and faster major-version upgrades. Plan prompt minor
updates and a roughly five-year major cadence.

## MVCC operations: bloat, vacuum, wraparound

- **Diagnose before tuning.** Check `n_dead_tup` vs `n_live_tup` in `pg_stat_user_tables` and
  `last_autovacuum`. Dead tuples piling up *despite autovacuum running* usually means something is
  holding back the visibility horizon. Check, in order:
  - `pg_stat_activity` for `state = 'idle in transaction'` and ancient `xact_start`;
  - `pg_replication_slots` for inactive slots;
  - long-running queries and orphaned prepared transactions (`pg_prepared_xacts`).
  Kill the holder; vacuum tuning was never the problem.
- **Autovacuum tuning reasoning:** defaults scale poorly for large hot tables —
  `autovacuum_vacuum_scale_factor = 0.2` means a 500M-row table waits for 100M dead tuples.
  Per-table override for such tables:
  `ALTER TABLE hot SET (autovacuum_vacuum_scale_factor = 0.01, autovacuum_vacuum_cost_delay = 1);`
  If autovacuum falls behind globally: raise `autovacuum_max_workers` **and**
  `autovacuum_vacuum_cost_limit` — workers share the cost budget, so more workers without more
  budget is the same total speed.
- **Wraparound is the one true emergency.** XIDs are 32-bit; as a table's oldest unfrozen XID
  ages toward ~2B, Postgres first forces aggressive vacuums, then refuses writes. Monitor
  `age(relfrozenxid)` and alert well before `autovacuum_freeze_max_age` (default 200M). The cause
  is almost always the same horizon-holders as above — abandoned replication slots and forgotten
  prepared transactions are the classic culprits. Remediate with `VACUUM (FREEZE)` on the oldest
  tables; do not restart-and-hope.
- **Existing bloat isn't removed by plain VACUUM** (it frees space for reuse; it doesn't shrink
  files). Use `pg_repack` online, or `VACUUM FULL` only with an accepted exclusive-lock outage.
  `REINDEX CONCURRENTLY` for bloated indexes.

## The planner conversation

Reading discipline for `EXPLAIN (ANALYZE, BUFFERS)`:
1. Find where **actual time** is spent — inner-most expensive node outward — not the biggest cost
   estimate.
2. Compare **estimated vs actual rows** at each node. Off by >10×: statistics problem. Run
   `ANALYZE`; then consider raising the column's statistics target; for correlated predicates the
   planner multiplies independent selectivities (`WHERE city='X' AND state='Y'` classic) — fix
   with extended statistics: `CREATE STATISTICS ... (dependencies) ON city, state FROM t;`
3. Read `BUFFERS`: huge `read` counts = cold cache or missing pruning; any `temp read/written` =
   `work_mem` too small for that sort/hash — raise it per-session for the offending query class,
   never globally (global work_mem multiplies by connections × nodes and eats RAM).

"Why is the planner ignoring my index?" — checklist in order of likelihood:
- **Selectivity:** the predicate matches a large fraction of the table; a seq scan is genuinely
  cheaper. The index was never going to be used, and that's correct.
- **Expression mismatch:** query says `WHERE lower(email) = ...` or `date_trunc('day', ts) = ...`
  but the index is on the bare column. Either create the matching expression index or rewrite the
  predicate sargably (`ts >= d AND ts < d + interval '1 day'`).
- **Type mismatch:** comparing a `varchar` column to a numeric literal, or joins across int4/int8;
  any cast applied to the *column* side defeats the index.
- **Leading-column rule:** a B-tree on `(a, b)` serves filters on `a` and on `a, b`; historically
  not `b` alone. PG18's skip scan softens this when `a` has few distinct values — treat it as a
  bonus, not a design assumption.
- **Stale stats after a bulk load:** autovacuum's ANALYZE hasn't fired yet; run `ANALYZE`
  explicitly after big loads.
- **Wrong index type for the operator:** `LIKE '%foo%'` needs a `pg_trgm` GIN index, not a B-tree;
  B-trees don't help leading-wildcard search no matter how much you want them to.

## Index taxonomy applied

- **B-tree** (default): equality and range on scalars; supports ORDER BY avoidance. The 95% answer.
- **GIN**: `jsonb` containment (`@>`), arrays, full-text (`tsvector`), trigram search. Expensive
  to update; brilliant to query. For write-heavy jsonb, consider indexing only the queried path
  via an expression index instead of the whole document.
- **GiST**: geometric/range overlap, KNN ordering, and exclusion constraints —
  `EXCLUDE USING gist (room WITH =, during WITH &&)` is the booking-overlap constraint you can't
  express with UNIQUE.
- **BRIN**: kilobytes to index terabytes of *physically correlated* data (append-only tables,
  timestamps). Useless once heavy updates shuffle the correlation — check `pg_stats.correlation`.
- **Partial**: `CREATE INDEX ... WHERE status = 'pending'` — index the 1% hot subset; smaller,
  cheaper to maintain. Also the soft-delete uniqueness pattern:
  `CREATE UNIQUE INDEX ... ON t (email) WHERE deleted_at IS NULL;`
- **Covering**: `INCLUDE (cols)` to earn index-only scans. Verify with low `Heap Fetches` in
  EXPLAIN — index-only scans depend on vacuum keeping the visibility map current.
- **Write-cost accounting:** every UPDATE writes N+1 index entries unless HOT-eligible, and HOT
  requires that *no indexed column changed*. Review `pg_stat_user_indexes.idx_scan` quarterly and
  drop zero-scan indexes (after checking replicas and rare batch jobs that might use them).

## Connection architecture (as of 2026)

- **Symptoms of the connection-per-client crisis:** hundreds-to-thousands of mostly idle
  connections, memory pressure, latency spikes under connection churn. Serverless/lambda apps are
  the worst offenders. Raising `max_connections` into the thousands degrades everyone.
- **PgBouncer modes:** `session` (1:1 while connected — safe, pools little), `transaction`
  (connection borrowed per transaction — the workhorse), `statement` (rare). Transaction mode
  breaks session state: `SET` (use `SET LOCAL`), advisory locks held across transactions,
  `LISTEN/NOTIFY`, temp tables, `WITH HOLD` cursors.
- **The prepared-statement gotcha:** protocol-level prepared statements (most drivers' default)
  historically broke in transaction mode — the next transaction may land on a server connection
  that never prepared the statement (`prepared statement "S_1" does not exist`). Modern PgBouncer
  (1.21+, as of 2026) supports them in transaction mode via `max_prepared_statements > 0`, which
  tracks and re-prepares per server connection. When you see missing-prepared-statement errors:
  set that, or disable driver-side prepares (JDBC `prepareThreshold=0`), or move that workload to
  session mode. SQL-level `PREPARE`/`EXECUTE` remains incompatible with transaction pooling.
- Managed poolers (RDS Proxy, Supavisor, Neon's pooler) inherit the same mode semantics — the
  transaction-mode caveats travel with the mode, not the product.
- **Sizing prior:** the optimal server-side pool is small — a few × CPU cores (classic OLTP
  starting point ≈ cores × 2–4) — not "as many as clients want". Queue in the pooler, not in
  the database.

## Locking diagnosis and migration discipline

- **The ALTER TABLE outage pattern:** `ALTER TABLE` needs `ACCESS EXCLUSIVE`. It waits behind any
  long query on the table — and because lock requests queue, every *new* `SELECT` then waits
  behind *it*. A 1ms DDL becomes a table-wide outage lasting as long as the longest running query.
  Discipline: run migrations with `SET lock_timeout = '2s'` — fail fast and retry, never
  queue-block — off-peak, with the app tolerating the retry.
- **Know the safe/unsafe DDL split:**
  - Brief-lock-only: `ADD COLUMN` (even with a constant default, PG11+), `DROP COLUMN`,
    `SET DEFAULT`.
  - Dangerous: `ALTER COLUMN TYPE` (usually a full rewrite — do the new-column/backfill/swap
    dance), `ADD COLUMN ... UNIQUE`, plain `CREATE INDEX` (blocks writes).
  - Always `CREATE INDEX CONCURRENTLY` on live tables. It cannot run inside a transaction, and a
    failure leaves an `INVALID` index behind — check for and drop it before retrying.
  - Split constraint addition: `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT`
    (short locks in both steps). `SET NOT NULL` cheaply by validating a CHECK constraint first
    (PG12+).
- **Diagnosing live contention:** join `pg_locks` to `pg_stat_activity`, or use
  `pg_blocking_pids(pid)` to walk to the blocker-chain root. The fix is usually terminating one
  idle-in-transaction session, not anything clever. Set `idle_in_transaction_session_timeout` so
  they die automatically.
- **Advisory locks** for application-level mutual exclusion (cron singletons, migration runners):
  `pg_try_advisory_lock(key)`. Session-scoped — so under transaction pooling use
  `pg_advisory_xact_lock` instead, or the lock outlives your logical session unpredictably.

## Extensions, replication, HA (as of 2026)

- **pgvector** (0.8.x): production-standard vector search inside Postgres. HNSW indexes with
  parallel builds (tune `m`, `ef_construction`, and query-time `hnsw.ef_search` for the
  recall/latency trade); `halfvec` halves storage for high-dimensional embeddings; IVFFlat only
  when build cost dominates. Verdict: for RAG/semantic search below tens of millions of vectors,
  pgvector removes the need for a dedicated vector database — one system, real transactions,
  joins with your own data.
- **PostGIS**: still the best geospatial engine anywhere. **Timescale-class time-series**
  (hypertables, compression, continuous aggregates) vs native declarative partitioning +
  `pg_partman`: choose native partitioning when you need pruning and retention-dropping; choose
  the time-series extension when you want compression and continuous aggregates managed for you.
  **pg_stat_statements** is non-negotiable on every production instance.
- **Streaming (physical) replication**: byte-level, whole-cluster, same major version; the HA
  backbone (with Patroni-class failover tooling). Lag consequence: read-your-writes violations on
  replicas — route session-critical reads to the primary or gate on LSN.
  `hot_standby_feedback = on` stops replica queries being cancelled by vacuum at the price of
  primary-side bloat — the MVCC horizon again.
- **Logical replication**: per-table, cross-version, cross-system — the mechanism for
  near-zero-downtime major upgrades and CDC into Kafka/warehouses (Debezium-class). Gotchas:
  DDL doesn't replicate (coordinate schema changes yourself; sequence handling improves in recent
  versions — verify against current docs before relying on it), and an *abandoned replication
  slot retains WAL until the disk fills*. Monitor slot lag, always.
- **Upgrades:** minors promptly (binary-compatible fixes). Majors via `pg_upgrade --link` (fast,
  brief downtime; PG18 additionally carries planner statistics across, cutting post-upgrade
  re-warm) or logical-replication switchover for near-zero downtime. `ANALYZE` after any upgrade
  path that loses statistics.

## How an expert thinks through it: "the app got slow this afternoon; CPU is fine"

*Slow with idle CPU = waiting, not computing. Waiting on locks, I/O, or connections. Cheapest
evidence first: `pg_stat_activity`.*

It shows 60 sessions `active` on the same UPDATE, and one session `idle in transaction` since
13:05. *Prior confirmed: a lock convoy behind an idle transaction — someone's script opened a
transaction, touched a row, and went to lunch.* `pg_blocking_pids` traces every waiter to that PID.

Considered and rejected:
- *Restart the database* — rejected: a sledgehammer that breaks everything else, and the pattern
  recurs tomorrow.
- *Raise max_connections because the pool is exhausted* — rejected: pool exhaustion is a symptom;
  more connections = more waiters in the same convoy.

`pg_terminate_backend(pid)` on the idler; the queue drains in seconds. Prevention:
`idle_in_transaction_session_timeout = '60s'`, find and fix the offending job's transaction scope,
and add monitoring on max transaction age.

*One more MVCC thought before closing:* that transaction was open for three hours — the vacuum
horizon was pinned all afternoon. Check `n_dead_tup` on hot tables and expect a bloat spike; let
autovacuum catch up or nudge the hottest tables with a manual `VACUUM`.

## Worked micro-example: making the planner use the (right) index

```sql
-- Query: recent pending jobs. Slow despite an index on (status).
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload FROM jobs
WHERE status = 'pending' AND created_at > now() - interval '1 hour'
ORDER BY created_at LIMIT 100;
-- Plan: Seq Scan; est rows 2,100,000 — status='pending' is 40% of the table.
-- Correct response is NOT "force the index": the status index is genuinely useless
-- at 40% selectivity. The hot subset is tiny and always queried the same way, so:

CREATE INDEX CONCURRENTLY jobs_pending_recent
    ON jobs (created_at)
    WHERE status = 'pending';
ANALYZE jobs;

-- Re-EXPLAIN: Index Scan using jobs_pending_recent; actual rows ≈ estimate; buffers tiny.
-- Partial index wins three ways: small (only pending rows), cheap to maintain
-- (completed rows never touch it), and shaped exactly like the query —
-- predicate in WHERE, range + ORDER BY on created_at.
```

## Verification / self-check

- Every performance claim is backed by an `EXPLAIN (ANALYZE, BUFFERS)` you actually ran — before
  and after — with estimated-vs-actual rows sanity-checked. Never "this should use the index".
- Every DDL statement you propose is annotated with its lock level and a `lock_timeout` strategy;
  `CREATE INDEX` says `CONCURRENTLY` or justifies why not.
- Every new index states its cost (write amplification, HOT loss) and the query it serves; every
  long-running process states its transaction scope (the MVCC-horizon check).
- Pooling advice names the PgBouncer mode and its session-state consequences explicitly.
- Version-sensitive claims (skip scan, uuidv7, pooler prepared-statement support) carry their
  version; anything unverified is phrased as principle, not as a flag name.
- Stopping rules: stop optimizing when the query meets its latency budget with correct plans at
  realistic data volume — not when EXPLAIN is maximally beautiful. Stop adding indexes when reads
  meet SLA; start *removing* them when `idx_scan = 0` for a quarter.
