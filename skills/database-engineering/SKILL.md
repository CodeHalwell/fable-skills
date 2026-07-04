---
name: database-engineering
description: Load when designing schemas, writing or optimizing SQL, choosing indexes, reading EXPLAIN plans, debugging slow queries or lock contention, planning migrations on live tables, or reasoning about transaction isolation, connection pools, or OLTP/OLAP boundaries.
---

# Database Engineering

## Core mental model

- **Indexes are sorted structures; queries either walk them in order or they don't.** A B-tree on `(a, b, c)` is sorted by `a`, then `b` within `a`, then `c` within `b`. It serves: equality on a prefix, then at most ONE range, then ordering by the next columns. The moment a range or missing column breaks the prefix, everything to its right is dead weight for seeking (still usable for filtering and covering).
- **Design indexes from query shape, not from columns that "seem important".** Collect the WHERE/JOIN/ORDER BY shapes first; the index follows mechanically (rule below). Indexing every column individually is a smell — single-column indexes rarely combine well, and each index taxes every write.
- **The optimizer is an estimator, not an oracle.** Bad plans are almost always bad row-count estimates: stale statistics, correlated columns, skewed values, or expressions that defeat statistics (`WHERE f(col) = x`). Read plans by hunting for the node where *estimated rows* diverges from *actual rows* by 10× or more — that's where the plan went wrong; everything downstream is a consequence.
- **Isolation levels are named by the anomalies they permit, and the defaults permit a lot.** Postgres and Oracle default to READ COMMITTED; MySQL/InnoDB defaults to REPEATABLE READ. Neither default prevents lost updates from read-modify-write in application code, and snapshot isolation (even when labeled "serializable", as in older Oracle) permits write skew. Know which anomaly your invariant needs blocked, then choose the cheapest level or lock that blocks it.
- **Row count × access frequency, not query complexity, determines cost.** A gnarly 6-way join over 10k rows is fine; a `SELECT ... WHERE status='pending'` scanning 200M rows 50×/sec is an outage. Always ask "how many rows does each node touch" before "how do I rewrite this SQL".
- **Locks are queues you can't see.** Every UPDATE takes row locks held until COMMIT; every DDL takes table locks. Slow queries are annoying; lock waits *cascade* — one long transaction holding a lock creates a convoy of waiters, each holding their own locks. Keep transactions short, and never hold one across a network call or user interaction.

## Decision frameworks

### Composite index column order (mechanical rule)

1. Equality predicates first (`WHERE tenant_id = ? AND status = ?`) — order among them barely matters for this one query; put the more reusable one first for prefix-sharing with other queries.
2. Then columns matching `ORDER BY` — lets the index deliver rows pre-sorted, which kills the Sort node AND enables early termination with `LIMIT`.
3. Then ONE range predicate (`created_at > ?`).
4. Then extra selected columns as index *includes* (Postgres `INCLUDE (...)`, or trailing key columns in MySQL) to make the index covering.

`WHERE tenant_id=? AND created_at>? ORDER BY created_at LIMIT 20` → `(tenant_id, created_at)`. Choosing `(created_at, tenant_id)` scans every tenant's recent rows — 100× worse under many tenants.

When ORDER BY and a *different* range column conflict, you can't have both in one B-tree walk. Rule of thumb: index for the ORDER BY + LIMIT (early termination) when the limit is small and matching rows are plentiful; index for the range when the range is highly selective. Check both plans if unsure.

### Covering index decision

- If a hot query reads 3–4 columns from a wide table via an index, add them as INCLUDE columns to skip heap/table lookups entirely (`Index Only Scan`).
- Worth it when the query runs constantly; not for rare queries — every included column bloats the index and slows writes.
- In Postgres, index-only scans also require a well-vacuumed table (visibility map). If `Heap Fetches` in EXPLAIN ANALYZE is high, the scan is index-only in name only — vacuum is behind.

### Isolation level / locking choice by invariant

| Invariant | Broken by | Cheapest fix |
|---|---|---|
| Counter/balance updated concurrently | Lost update under READ COMMITTED (two clients read 100, both write 90) | Atomic write: `UPDATE ... SET x = x - 5` — never read-then-write |
| Read-modify-write with app logic between | Same lost update | `SELECT ... FOR UPDATE`, or optimistic version column (`UPDATE ... WHERE id=? AND version=?`, check rowcount, retry on 0) |
| "At least one on-call doctor" / sum-across-rows constraints | Write skew under snapshot isolation: two txns each read the invariant as safe, write *different* rows, both commit | True SERIALIZABLE (Postgres SSI — must catch and retry serialization failures), or materialize the constraint into one row and update it, forcing a write-write conflict |
| Uniqueness | Check-then-insert race at any level below serializable | UNIQUE constraint + handle the violation; never SELECT-then-INSERT |
| Referenced row must exist | Concurrent delete of the parent | FK constraint, or `FOR SHARE` lock on the parent row |

Choosing between pessimistic (`FOR UPDATE`) and optimistic (version column): pessimistic when conflicts are common or retry is expensive; optimistic when conflicts are rare and you can retry cheaply. Optimistic locking also survives connection pools and multi-request workflows where holding a row lock is impossible.

### Migration safety on hot tables (Postgres specifics; the reasoning ports)

- `CREATE INDEX CONCURRENTLY` — plain CREATE INDEX blocks writes for the whole build. Caveats: concurrent build can't run inside a transaction and can leave an INVALID index on failure — check `pg_index.indisvalid`, drop and retry.
- `ALTER TABLE ... ADD COLUMN x type DEFAULT constant` is instant in modern Postgres. Adding a column with a *volatile* default, or changing a column's type, rewrites the whole table under ACCESS EXCLUSIVE — do it as add-nullable-column → batched backfill → add constraint.
- Adding NOT NULL or CHECK: `ADD CONSTRAINT ... NOT VALID` (instant) then `VALIDATE CONSTRAINT` (weak lock, full scan) — never a direct validated add on a big table.
- **Every ALTER needs `SET lock_timeout = '2s'` first.** Even a "fast" ALTER needs a brief ACCESS EXCLUSIVE lock; if it queues behind one long-running query, every subsequent query queues behind *it* — a 3-second ALTER becomes a site-wide stall. Fail fast and retry in a loop instead.
- Backfills: batched (`UPDATE ... WHERE id BETWEEN ...` in 1–10k-row chunks, with sleeps), never one giant UPDATE — long transactions block vacuum, bloat the table, and hold locks for the duration.
- Deploy ordering: adding a column the app writes → migrate first, deploy second. Dropping → deploy code that stops using it first, drop second. Renaming → never directly; add new column, dual-write, backfill, switch reads, drop old. A deployed app must never reference a column that might not exist.

### Normalization judgment

- Normalize by default (3NF-ish): update anomalies are correctness bugs you can't index your way out of.
- Denormalize only when all three hold: (a) a read path is hot AND requires a join/aggregate you've *measured* as the bottleneck; (b) the duplicated data changes rarely, or you have a reliable transactional/async updater; (c) you write down which copy is the source of truth.
- Precomputed aggregates (`order_count` on users) are the most common justified case — maintain them in the same transaction as the source rows, or by trigger, never "the app remembers to".
- JSON columns are denormalization too: fine for genuinely schemaless payloads you never filter on; wrong for attributes you query, join, or constrain — those want columns.

### OLTP vs OLAP boundary

- The moment a query aggregates over more than ~5–10% of a large table, or scans months of history, it doesn't belong on the OLTP primary — it evicts the hot working set from the buffer pool and its long snapshot holds back vacuum.
- Escalation ladder: read replica (cheap, minutes to set up; watch replication lag) → nightly extract to a warehouse → CDC-fed columnar store (ClickHouse/BigQuery/Snowflake) when freshness matters.
- Row stores read whole rows; analytics reads few columns of many rows — that's why columnar wins by 10–100× there, and why "just add an index" doesn't fix analytic queries: no B-tree helps a full aggregation.

### Connection pool sizing

- Connections are expensive server-side (Postgres: one process each, real memory, contention on shared structures). Optimal pool size ≈ `cores × 2 + effective IO parallelism` — for a typical box, tens, not thousands. A 4,000-connection pile-up makes everything slower, not more parallel.
- App instances × per-instance pool must stay under `max_connections` with headroom for admin/cron. When instance count × pool overflows, put PgBouncer in transaction-pooling mode in front — but transaction pooling breaks session state: session-level prepared statements (on older PgBouncer), advisory locks, `SET` parameters, temp tables.
- Symptom table: threads waiting on pool checkout while the DB sits idle → pool too small. DB CPU pegged with plenty of free pool slots → pool too big; shrink it and let requests queue in the app, where waiting is cheap.

## Failure modes & pitfalls

- **N+1 queries.** One query for a list, then one query per item — 1+N round trips. Detect: query log shows the same statement shape with different params in a tight burst; ORM lazy-loading is the usual cause (`for order in orders: order.customer.name`). Fix: eager load (`selectinload`/`joinedload` in SQLAlchemy, `select_related`/`prefetch_related` in Django, `include` in Prisma) or one `WHERE id IN (...)` batch. The 10ms-per-query version passes tests with 3 rows and times out with 3,000.
- **Function on an indexed column kills the index.** `WHERE DATE(created_at) = '2026-07-04'` and `WHERE lower(email) = ?` can't use a plain index on the column. Rewrite as a range (`created_at >= d AND created_at < d + interval '1 day'`) or create the expression index (`CREATE INDEX ON users (lower(email))`). Same trap in disguise: comparing a varchar column to an integer parameter — the implicit cast lands on the column side.
- **Leading-wildcard LIKE.** `LIKE '%foo%'` can't use a B-tree; it needs a `pg_trgm` GIN index or full-text search. `LIKE 'foo%'` is fine — it's a range scan.
- **OFFSET pagination on big tables.** `OFFSET 100000 LIMIT 20` reads and discards 100k rows, getting linearly slower per page, and skips/duplicates rows under concurrent inserts. Use keyset pagination: `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20` with a matching index.
- **Lost update shipped as "we use transactions".** `BEGIN; SELECT balance; <app computes>; UPDATE balance = <computed>; COMMIT` under READ COMMITTED loses concurrent updates — wrapping a read-modify-write in a transaction does not serialize it. Fix per the isolation table. This is the single most common real-world isolation bug.
- **Write skew passes code review because each transaction "checks first".** Two on-call doctors both run "count other on-call → ≥1 → remove myself". Under snapshot isolation both reads see the old state; both commit; zero on-call. The tell: a check-then-write where the check reads rows the write doesn't touch.
- **CTE as optimization fence.** In Postgres ≤11 always, and in 12+ when a CTE is referenced more than once or written `AS MATERIALIZED`, the CTE materializes fully — outer predicates don't push in, so `WITH big AS (SELECT * FROM events) SELECT * FROM big WHERE user_id=?` scans all of `events`. Inline it, or verify the plan shows the filter inside the scan.
- **Window function filtered in the wrong place.** Window results can't appear in WHERE (they're computed after it). `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC)` must be wrapped in a subquery and filtered outside (`WHERE rn = 1`). Also: window functions don't reduce rows — mixing them with GROUP BY intuitions double-counts.
- **EXPLAIN without ANALYZE.** Plain EXPLAIN shows estimates only; a plan can look fine while actuals are catastrophic. Use `EXPLAIN (ANALYZE, BUFFERS)` — inside a transaction you roll back, for writes. `Rows Removed by Filter: 4,999,832` means the index isn't doing the selection you think it is.
- **Nested loop chosen from a bad estimate.** Estimate says 3 rows, actual is 300k → the inner index scan executes 300k times. Fix the *estimate* — `ANALYZE` the table, raise the column's statistics target, avoid expressions on columns, add extended statistics (`CREATE STATISTICS`) for correlated columns — rather than reaching for join hints.
- **Index added, writes got slower, and the index isn't even used.** Every index is maintained on every INSERT and on every UPDATE touching its columns, and an update to any indexed column disables Postgres HOT updates for that row. Audit with `pg_stat_user_indexes.idx_scan = 0` before and after adding "just in case" indexes.
- **`ORDER BY x LIMIT 1` without an index on x → full sort of the table.** With the index it's a single probe at one end. The difference hides until the table grows past memory.
- **Long-running transaction as silent saboteur.** An idle-in-transaction connection (forgotten commit, debugger attached, ORM session leak) blocks vacuum from reclaiming any row version newer than its snapshot → table and index bloat, plans degrade fleet-wide. Alert on `pg_stat_activity` transactions older than a few minutes; set `idle_in_transaction_session_timeout`.
- **NULL logic in NOT IN.** `WHERE id NOT IN (SELECT ref_id FROM t)` returns zero rows if any `ref_id` is NULL (three-valued logic: `x <> NULL` is unknown). Use `NOT EXISTS`, which has no such trap and usually plans better.

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

Read it inside-out:
1. The index serves only `customer_id`; 48k rows come back from it.
2. 46k of them are thrown away by the `status` filter — applied on the heap, after 39,544 buffer reads.
3. The ~2k survivors are fully sorted just to keep 10.

Fix, straight from the mechanical rule (equalities → order-by): `CREATE INDEX CONCURRENTLY idx_orders_cust_status_created ON orders (customer_id, status, created_at DESC);`

New plan: `Index Scan using idx_orders_cust_status_created ... rows=10`, no Sort node, ~10 buffer reads — it walks the index in output order and stops after 10 rows. Adding `INCLUDE (total)` makes it index-only (`id` comes free from the key) — but verify `Heap Fetches: 0`, else run VACUUM. 1.8s → sub-millisecond, and every step was mechanical from the query shape.

## Verification / self-check

- For any index recommendation: state the exact query shape it serves; walk the prefix rule (equalities → ORDER BY → one range); confirm no function/cast wraps an indexed column; name which existing index it makes redundant (a prefix of the new one) and drop it.
- For any slow-query diagnosis: produce or demand `EXPLAIN (ANALYZE, BUFFERS)`; identify the node where estimate vs actual diverges ≥10×; your fix must change *that node*. If you can't point at the node, you're guessing.
- For any concurrency claim: name the anomaly (lost update / write skew / phantom / non-repeatable read), state the isolation level actually in effect (check the engine's default — don't assume), and simulate two interleaved transactions by hand, statement by statement.
- For any migration: state the lock each statement takes, what it blocks, and for how long; confirm `lock_timeout` is set and the code-deploy ordering is explicit.
- For any capacity claim, do the arithmetic: rows touched × frequency × bytes/row vs buffer pool size; connections × work_mem vs RAM. If the numbers aren't written down, the claim isn't checked.
