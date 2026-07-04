---
name: database-engineering
description: Load when designing schemas, writing or optimizing SQL, choosing indexes, reading EXPLAIN plans, debugging slow queries or lock contention, planning migrations on live tables, or reasoning about transaction isolation, connection pools, or OLTP/OLAP boundaries.
---

# Database Engineering

## Core mental model

- **Indexes are sorted structures; queries either walk them in order or they don't.** A B-tree on `(a, b, c)` is sorted by `a`, then `b` within `a`, then `c` within `b`. It serves: equality on a prefix, then at most ONE range, then ordering by the next columns. The moment a range or missing column breaks the prefix, everything to its right is dead weight for seeking (still usable for filtering/covering).
- **Design indexes from query shape, not from columns that "seem important".** Collect the WHERE/JOIN/ORDER BY shapes first; the index follows mechanically (see rule below). Indexing every column individually is a smell — single-column indexes rarely combine well, and each index taxes every write.
- **The optimizer is an estimator, not an oracle.** Bad plans are almost always bad row-count estimates: stale statistics, correlated columns, skewed values, or expressions that defeat statistics (`WHERE f(col) = x`). Read plans by hunting for the node where *estimated rows* diverges from *actual rows* by 10× or more — that's where the plan went wrong; everything downstream is a consequence.
- **Isolation levels are named by the anomalies they permit, and the defaults permit a lot.** Postgres/Oracle default READ COMMITTED, MySQL/InnoDB default REPEATABLE READ. Neither default prevents lost updates from read-modify-write in application code, and even SERIALIZABLE-labeled snapshot isolation (older Oracle "serializable") permits write skew. Know which anomaly your invariant needs blocked, then choose the cheapest level or lock that blocks it.
- **Row count × access frequency, not query complexity, determines cost.** A gnarly 6-way join over 10k rows is fine; a `SELECT ... WHERE status='pending'` scanning 200M rows 50×/sec is an outage. Always ask "how many rows does each node touch" before "how do I rewrite this SQL".

## Decision frameworks

**Composite index column order (mechanical rule):**
1. Equality predicates first (`WHERE tenant_id = ? AND status = ?`) — order among them barely matters for this query; put the more reusable/selective one first for prefix sharing with other queries.
2. Then columns matching `ORDER BY` (lets the index deliver rows pre-sorted — kills the sort node AND enables early termination with `LIMIT`).
3. Then ONE range predicate (`created_at > ?`).
4. Then extra selected columns as index *includes* (Postgres `INCLUDE (...)`, or trailing key columns in MySQL) to make it covering.

`WHERE tenant_id=? AND created_at>? ORDER BY created_at LIMIT 20` → `(tenant_id, created_at)`. Choosing `(created_at, tenant_id)` scans every tenant's recent rows — 100× worse under many tenants. Rule of thumb when ORDER BY and a *different* range column conflict: index for the ORDER BY + LIMIT (early termination) when the limit is small and matching rows are plentiful; index for the range when it's highly selective.

**Covering index decision:** if a hot query reads 3–4 columns from a wide table via an index, add them as INCLUDE columns to skip the heap/table lookups entirely (`Index Only Scan`). Worth it when the query runs constantly; not worth it for rare queries — every included column bloats the index and slows writes. In Postgres, index-only scans also require a reasonably vacuumed table (visibility map); if `Heap Fetches` in EXPLAIN ANALYZE is high, vacuum is behind.

**Isolation level / locking choice by invariant:**

| Invariant | Broken by | Cheapest fix |
|---|---|---|
| Counter/balance updated concurrently | Lost update under READ COMMITTED (two clients read 100, both write 90) | Atomic write: `UPDATE ... SET x = x - 5` — never read-then-write |
| Read-modify-write with logic between | Same lost update | `SELECT ... FOR UPDATE`, or optimistic version column (`UPDATE ... WHERE version = ?`, check rowcount) |
| "At least one on-call doctor" / sum-across-rows constraints | Write skew under snapshot isolation: two txns each read the invariant as safe, write different rows, both commit | True SERIALIZABLE (Postgres SSI — must retry serialization failures), or materialize the constraint into one row and update it (forces a write-write conflict) |
| Uniqueness | Check-then-insert race at any level below serializable | UNIQUE constraint + handle the violation; never SELECT-then-INSERT |
| Foreign object must exist | Concurrent delete | FK constraint, or FOR SHARE lock on the parent |

**Migration safety on hot tables (Postgres specifics; the reasoning ports):**
- `CREATE INDEX CONCURRENTLY` — plain CREATE INDEX blocks writes for the whole build. (Concurrent build can't run in a transaction and can leave an INVALID index on failure — check and drop/retry.)
- `ALTER TABLE ... ADD COLUMN x type DEFAULT val` is instant in modern Postgres for constant defaults; adding a column with a *volatile* default, or changing a column's type, rewrites the table under an exclusive lock — do it as add-column → backfill in batches → swap.
- Adding NOT NULL or a CHECK: `ADD CONSTRAINT ... NOT VALID` (instant) then `VALIDATE CONSTRAINT` (weak lock, full scan) — never a direct validated add.
- **Every ALTER needs `lock_timeout`** (e.g., `SET lock_timeout = '2s'`). Even a "fast" ALTER needs a brief ACCESS EXCLUSIVE lock; if it queues behind one long-running query, every subsequent query queues behind *it* — a 3-second ALTER becomes a site-wide stall. Fail fast and retry instead.
- Backfills: batched (`UPDATE ... WHERE id BETWEEN ...` in 1–10k-row chunks with sleeps), never one giant UPDATE — long transactions block vacuum, bloat the table, and hold locks.
- Deploy order for adding a column the app writes: migrate first, then deploy code. For dropping: deploy code that stops using it, then drop. Never let a deployed app reference a column that may not exist on some replica/instance.

**Normalization judgment:** normalize by default (3NF-ish) because update anomalies are bugs you can't index your way out of. Denormalize only when: (a) a read path is hot AND requires a join/aggregate you've measured as the bottleneck, (b) the duplicated data changes rarely or you have a reliable async updater, and (c) you write down what the source of truth is. Precomputed aggregates (`order_count` on users) are the most common justified case — maintain them in the same transaction as the source rows or via triggers, not "the app remembers to".

**OLTP vs OLAP boundary:** the moment a query aggregates over >~5–10% of a large table, or scans months of history, it doesn't belong on the OLTP primary — it evicts the hot working set from cache and holds back vacuum. Ladder: read replica (cheap, minutes to set up) → nightly extract to a warehouse → CDC-fed columnar store (ClickHouse/BigQuery/Snowflake) when freshness matters. Row stores read whole rows; analytics reads few columns of many rows — that's why columnar wins by 10–100× there, and why "just add an index" doesn't fix analytic queries.

**Connection pool sizing:** connections are expensive server-side (Postgres: a process each; memory; contention). Optimal pool ≈ `cores * 2 + effective_spindle/IO concurrency` — for a typical Postgres box, tens, not thousands. A 4000-connection pile-up makes everything slower, not more parallel. App instances × per-instance pool must stay under `max_connections` with headroom; when instance count × pool overflows, put PgBouncer (transaction pooling mode) in front — but transaction pooling breaks session state: prepared statements (older PgBouncer), advisory locks, `SET`, temp tables. Sizing symptom table: threads waiting on pool checkout + idle DB → pool too small; DB CPU saturated → pool too big (queue in the app instead, where waiting is cheap).

## Failure modes & pitfalls

- **N+1 queries.** Symptom: one query for a list, then one query per item — 1+N round trips. Detect: query log shows the same statement shape with different params in a tight burst; ORM lazy-loading is the usual cause (`for order in orders: order.customer.name`). Fix: eager load (`selectinload`/`joinedload` in SQLAlchemy, `include` in Prisma, `select_related`/`prefetch_related` in Django) or one `WHERE id IN (...)` batch. The 10ms-per-query version passes tests with 3 rows and times out with 3000.
- **Function on an indexed column kills the index.** `WHERE DATE(created_at) = '2026-07-04'` and `WHERE lower(email) = ?` can't use the plain index. Rewrite as a range (`created_at >= date AND created_at < date + 1`) or create the expression index (`CREATE INDEX ON users (lower(email))`). Same trap: comparing a varchar column to an integer (implicit cast on the column side).
- **Leading-wildcard LIKE.** `LIKE '%foo%'` can't use a B-tree. Needs `pg_trgm` GIN index or full-text search. `LIKE 'foo%'` is fine (it's a range).
- **OFFSET pagination on big tables.** `OFFSET 100000 LIMIT 20` reads and discards 100k rows, getting linearly slower per page, and skips/duplicates rows under concurrent inserts. Use keyset pagination: `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20` with a matching index.
- **Lost update shipped as "we use transactions".** `BEGIN; SELECT balance; ... app computes ...; UPDATE balance = <computed>; COMMIT` under READ COMMITTED loses concurrent updates — transactions don't serialize your read-modify-write. Fix per the isolation table above. This is the single most common real-world isolation bug.
- **Write skew passes code review because each transaction "checks first".** Two on-call doctors both run "count on-call others → ≥1 → take myself off". Under snapshot isolation both reads see the old state; both commit; zero on-call. The check-then-write pattern across *different rows* is the tell.
- **CTE as optimization fence.** In Postgres ≤11, and in 12+ when a CTE is referenced multiple times or written `AS MATERIALIZED`, the CTE materializes fully — predicates from outside don't push in, so `WITH big AS (SELECT * FROM events) SELECT * FROM big WHERE user_id=?` scans all of `events`. Inline it or check the plan shows the filter inside the scan.
- **Window function filtered in the wrong place.** You can't put a window result in WHERE (it's computed after WHERE). `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` must be wrapped in a subquery and filtered outside (`WHERE rn = 1`). Also: window functions don't reduce rows — mixing them with expectations of GROUP BY semantics double-counts.
- **EXPLAIN without ANALYZE.** Plain EXPLAIN shows estimates only; the plan may look fine while actuals are catastrophic. Use `EXPLAIN (ANALYZE, BUFFERS)` (in a transaction you roll back, for writes). Read it inside-out: find the deepest node where actual rows ≫ estimated rows, and look at `Buffers:` for the real I/O. `Rows Removed by Filter: 4,999,832` means the index isn't doing the selection you think it is.
- **Nested loop chosen from a bad estimate.** Estimate says 3 rows, actual is 300k → nested loop executes the inner index scan 300k times. Fix the estimate (ANALYZE the table, raise the column's statistics target, avoid expressions on columns, extended statistics for correlated columns) rather than hinting the join.
- **Index added, writes got slow, and the index isn't even used.** Every index is maintained on every INSERT/UPDATE of its columns and disables Postgres HOT updates when an indexed column changes. Audit with `pg_stat_user_indexes.idx_scan = 0` before adding "just in case" indexes.
- **`ORDER BY x LIMIT 1` without index → full sort of the table.** With an index on `x` it's one probe. The difference hides until the table grows.

## Worked micro-example

Query: `SELECT id, total FROM orders WHERE customer_id = 42 AND status = 'shipped' ORDER BY created_at DESC LIMIT 10;`

`EXPLAIN (ANALYZE, BUFFERS)` shows:

```
Limit (actual time=1840.2..1840.2 rows=10)
  -> Sort (actual time=1840.1..1840.1 rows=10)
       Sort Key: created_at DESC
       -> Bitmap Heap Scan on orders (actual rows=48211)
            Recheck Cond: (customer_id = 42)
            Rows Removed by Filter: 46102   -- status filter applied on heap
            Buffers: shared read=39544
            -> Bitmap Index Scan on idx_orders_customer (actual rows=48211)
```

Diagnosis, in order: the index serves only `customer_id`; 48k rows are fetched from the heap, 46k thrown away by the `status` filter; the survivors are sorted just to keep 10. Fix: `CREATE INDEX CONCURRENTLY idx_orders_cust_status_created ON orders (customer_id, status, created_at DESC);` — equality columns first, ORDER BY column last. New plan: `Index Scan ... rows=10`, no Sort node, ~10 buffer reads: it walks the index in output order and stops after 10 rows. Adding `INCLUDE (total)` makes it index-only (id comes free — but verify `Heap Fetches: 0`, else run VACUUM). 1.8s → sub-millisecond, and the reasoning was entirely mechanical from the query shape.

## Verification / self-check

- For any index recommendation: state the exact query shape it serves, walk the prefix rule (equalities → order → one range), and confirm no function/cast wraps an indexed column.
- For any slow-query diagnosis: demand or produce `EXPLAIN (ANALYZE, BUFFERS)`; identify the node where estimate vs actual diverges ≥10×; your fix must change *that node*.
- For any concurrency claim: name the anomaly (lost update / write skew / phantom), the isolation level in play (check the DB's default — don't assume), and simulate two interleaved transactions by hand.
- For any migration: state the lock each statement takes, what it blocks, and for how long; confirm `lock_timeout` is set and the code-deploy ordering is stated.
- For row-count claims, sanity-check with arithmetic: rows touched × frequency × bytes/row against the buffer pool size — does the working set fit?
