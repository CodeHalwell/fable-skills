---
name: postgres-mastery
description: Load for PostgreSQL work beyond basic CRUD — query performance and EXPLAIN analysis, index selection, vacuum/bloat/autovacuum issues, connection pooling, lock contention or migration outages, replication choices, extension selection (pgvector/PostGIS/Timescale-class), or Postgres version/upgrade decisions.
---

# PostgreSQL Mastery

Audit finding: frontier models hold nearly all operational Postgres knowledge cold — MVCC/horizon diagnosis, PG18 features, planner checklists, PgBouncer prepared-statement support, pgvector tuning, migration lock discipline. This file is therefore a dense checklist for completeness during reviews, not instruction. Trust the model's derivations; use this to make sure nothing is skipped.

## Version anchor (2026)

PostgreSQL 18 (Sept 2025): async I/O subsystem, native `uuidv7()` (prefer over v4 for B-tree keys), B-tree skip scan (bonus, not a design assumption), virtual generated columns default, temporal PK/FK, `OLD`/`NEW` in `RETURNING`, `pg_upgrade` carries planner statistics. PG19 beta summer 2026. Minor updates promptly; ~5-year major cadence.

## MVCC / vacuum checklist

- Dead tuples piling up despite autovacuum → find the horizon-holder first, in order: `idle in transaction` sessions (ancient `xact_start`), inactive replication slots, long queries, `pg_prepared_xacts`, `hot_standby_feedback`. Kill the holder; vacuum tuning was never the problem.
- Large hot tables: per-table `autovacuum_vacuum_scale_factor = 0.01` (or 0 + fixed threshold), `cost_delay = 1`; raising `autovacuum_max_workers` without raising `autovacuum_vacuum_cost_limit` splits the same budget — same total speed.
- Wraparound: monitor `age(relfrozenxid)`, alert well before `autovacuum_freeze_max_age` (200M); culprits = the same horizon-holders; remediate with `VACUUM (FREEZE)` on the oldest tables, never restart-and-hope.
- Plain VACUUM never shrinks files: `pg_repack` online, `VACUUM FULL` only with an accepted outage, `REINDEX CONCURRENTLY` for bloated indexes.
- Long-open transactions pin the horizon database-wide — after fixing any convoy/lock incident, check `n_dead_tup` on hot tables and expect a bloat spike.

## Planner checklist (`EXPLAIN (ANALYZE, BUFFERS)`)

- Actual time innermost-out; est-vs-actual rows off >10× = stats (ANALYZE, statistics target, `CREATE STATISTICS (dependencies)` for correlated predicates); `temp read/written` = per-session `work_mem`, never global.
- Index ignored, in likelihood order: correct low-selectivity judgment / expression mismatch (index `lower(email)` or rewrite sargably) / type mismatch (cast on column side) / leading-column rule (skip scan is a PG18 bonus) / stale stats after bulk load / wrong index type (`%foo%` needs `pg_trgm` GIN).
- Index taxonomy one-liners: B-tree 95%; GIN for jsonb/arrays/FTS/trigram (write-expensive — consider expression index on the queried path); GiST for overlap/KNN/exclusion constraints (`EXCLUDE USING gist (room WITH =, during WITH &&)`); BRIN only while physical correlation holds; partial for hot subsets and soft-delete uniqueness (`WHERE deleted_at IS NULL`); covering `INCLUDE` for index-only scans (verify Heap Fetches — depends on vacuum/visibility map).
- Write cost: every UPDATE writes N+1 index entries unless HOT (no indexed column changed). Quarterly: drop `idx_scan = 0` indexes (check replicas first).

## Connections

- Pool always (PgBouncer-class); server-side pool ≈ cores × 2–4, queue in the pooler; raising `max_connections` makes convoys worse.
- Transaction mode breaks: session `SET` (use `SET LOCAL`), session advisory locks (use `pg_advisory_xact_lock`), `LISTEN/NOTIFY`, temp tables, `WITH HOLD` cursors. Protocol-level prepared statements work since PgBouncer 1.21 via `max_prepared_statements > 0`; SQL-level `PREPARE` still doesn't. Managed poolers (RDS Proxy, Supavisor, Neon) inherit the mode's caveats.

## Migration / lock discipline

- `ALTER TABLE` takes ACCESS EXCLUSIVE and *queues*: a 1ms DDL behind one long query blocks every new SELECT behind it. Always `SET lock_timeout = '2s'` + retry, off-peak.
- Safe: `ADD COLUMN` (constant default PG11+), `DROP COLUMN`, `SET DEFAULT`, `VALIDATE CONSTRAINT`. Dangerous: `ALTER COLUMN TYPE` (rewrite — new-column/backfill/swap), inline UNIQUE/PK, plain `CREATE INDEX`.
- `CREATE INDEX CONCURRENTLY`: not in a transaction; failure leaves an INVALID index — check and drop before retry. Constraints: `NOT VALID` then `VALIDATE`; `SET NOT NULL` via validated CHECK (PG12+); PK via `CREATE UNIQUE INDEX CONCURRENTLY` + `ADD CONSTRAINT ... USING INDEX`.
- Contention diagnosis: `pg_blocking_pids()` to the chain root; fix is usually terminating one idle-in-transaction session + `idle_in_transaction_session_timeout`. Slow-app-idle-CPU = waiting (locks, pool, I/O), and `pg_stat_activity` is the cheapest evidence; rejected reflexes: restart (recurs tomorrow), raise max_connections (more waiters in the same convoy).

## Extensions / replication anchors (2026)

- pgvector 0.8.x: HNSW default (`m`, `ef_construction`, query-time `hnsw.ef_search`; `halfvec` halves storage; IVFFlat only when build cost dominates). Below tens of millions of vectors, one system with real transactions beats a dedicated vector DB.
- pg_stat_statements non-negotiable. Timescale-class vs native partitioning + `pg_partman`: choose the extension for compression + continuous aggregates, native for pruning/retention-dropping.
- Physical replication = HA backbone (Patroni-class); replica lag → read-your-writes violations (route session-critical reads to primary or gate on LSN); `hot_standby_feedback = on` trades replica cancellations for primary-side bloat.
- Logical replication = cross-version upgrades + CDC; DDL doesn't replicate; an abandoned slot retains WAL until the disk fills — monitor slot lag, always.
- Majors: `pg_upgrade --link` (PG18 keeps stats) or logical switchover; `ANALYZE` after any path that loses stats.

## Worked micro-example (kept for the shape of the argument)

```sql
-- 'pending' is 40% of jobs → the status index is correctly ignored; don't force it.
-- The hot subset is tiny and always queried the same way:
CREATE INDEX CONCURRENTLY jobs_pending_recent
    ON jobs (created_at) WHERE status = 'pending';
-- Small (only pending rows), cheap to maintain, and shaped exactly like the query:
-- predicate in WHERE, range + ORDER BY on created_at.
```

## Verification / self-check

- Every performance claim backed by an EXPLAIN (ANALYZE, BUFFERS) you ran, before and after, est-vs-actual checked.
- Every DDL annotated with lock level + lock_timeout strategy; every index states its write cost and its query.
- Pooling advice names the mode and its session-state consequences.
- Version-sensitive claims carry their version; unverified → phrase as principle, not flag name.
- Stop optimizing at the latency budget with correct plans at realistic volume; stop adding indexes at read SLA; start removing at `idx_scan = 0` for a quarter.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 12 baseline, 0 partial, 0 delta — the strongest baseline of the audited set.
- Opus cold nails: horizon-holder diagnosis ordering, all PG18 headline features, planner checklist, workers/cost-limit shared budget, PgBouncer 1.21 `max_prepared_statements`, NOT VALID/CHECK-then-NOT-NULL tricks, the exact partial-index answer, pgvector HNSW tuning + scale verdict, wraparound remediation, cores×2 pool sizing.
- Restructured from survey to review checklist; this skill's remaining job is completeness under pressure, not teaching.
