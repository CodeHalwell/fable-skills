---
name: database-engineering
description: Load when designing schemas, writing or optimizing SQL, choosing indexes, reading EXPLAIN plans, debugging slow queries or lock contention, planning migrations on live tables, or reasoning about transaction isolation, connection pools, or OLTP/OLAP boundaries.
---

# Database Engineering

Compact checklist. The standard expert answers — E-S-R index ordering, `EXPLAIN (ANALYZE, BUFFERS)` estimate-vs-actual hunting, lost-update and write-skew fixes, `CREATE INDEX CONCURRENTLY` + `NOT VALID`/`VALIDATE` + fast defaults + `lock_timeout`, keyset pagination, `NOT IN` NULL poisoning, CTE fences, pool sizing ≈ cores×2, expand/contract deploy ordering — are assumed known and appear only as anchors.

## Anchors (apply mechanically, don't re-derive)

- Composite index: equalities → ORDER BY columns → ONE range → INCLUDE the selected columns. ORDER-BY-vs-range conflict: small LIMIT with plentiful matches → index for the ORDER BY (early termination); highly selective range → index for the range. Check both plans if unsure.
- Among equality columns, ordering doesn't matter *for this query* — choose by **prefix reusability across other queries**, not by "most selective first" (that folk rule buys nothing within one query and costs shared prefixes).
- Slow-query method: `EXPLAIN (ANALYZE, BUFFERS)`; find the node where estimate vs actual diverges ≥10×; fix *that* (ANALYZE, statistics target, extended statistics for correlated columns, remove expressions from columns) — not join hints.
- Isolation: Postgres/Oracle default READ COMMITTED; MySQL REPEATABLE READ. Wrapping read-modify-write in a transaction does NOT serialize it. Fix ladder: atomic `SET x = x - 5` → `FOR UPDATE` → version-column optimistic (rowcount check + retry) → SERIALIZABLE with 40001 retry. Write skew (check reads rows the write doesn't touch): SERIALIZABLE, or materialize the constraint into one row.
- Migrations on hot tables: `SET lock_timeout='2s'` before EVERY ALTER (a queued ACCESS EXCLUSIVE stalls the whole table behind it); `CREATE INDEX CONCURRENTLY` (can leave INVALID index — check `indisvalid`, drop, retry); constant defaults instant (PG11+), volatile defaults rewrite; `NOT VALID` then `VALIDATE`; batched backfills; add = migrate-then-deploy, drop = deploy-then-migrate, rename = never (add → dual-write → backfill → switch → drop).
- Pool too small: threads queue on checkout while DB idles. Pool too big: DB CPU pegged with free slots — shrink and let requests queue in the app. PgBouncer transaction mode breaks session state (prepared statements pre-1.21, advisory locks, SET, temp tables).

## Sharpened corrections (where standard answers are thin)

- **Indexes tax writes twice, and the second tax is invisible: HOT updates.** Adding an index on a column means any UPDATE touching that column can no longer be a HOT (heap-only-tuple) update for that row — every such update now maintains *all* indexes on the table, not just the new one. Symptom: "we added one index and unrelated write throughput dropped." Audit candidates with `pg_stat_user_indexes.idx_scan = 0` before and after; drop indexes that a new index's prefix makes redundant.
- **Long-running/idle-in-transaction connections are fleet-wide saboteurs**, not local bugs: they pin the vacuum horizon, so *every* table accumulates dead versions, index-only scans regress (visibility map decays, `Heap Fetches` climbs), and plans degrade globally. Alert on `pg_stat_activity` transactions older than minutes; set `idle_in_transaction_session_timeout`. When index-only scans "stop working," suspect this before suspecting the index.
- **OLTP/OLAP boundary rule of thumb:** a query aggregating over more than ~5–10% of a large table, or scanning months of history, doesn't belong on the primary — it evicts the hot working set and holds back vacuum. Ladder: read replica → nightly extract → CDC-fed columnar store. No B-tree fixes a full aggregation; that's a column-store problem.

## Pitfall one-liners (audit on review)

- N+1 (same statement shape in a tight burst; ORM lazy loads) → eager load / `IN` batch.
- Function or implicit cast on an indexed column; leading-wildcard LIKE (needs pg_trgm).
- OFFSET pagination → keyset with row-value comparison `(a,b) < (?,?)` + matching index.
- Window functions can't be filtered in WHERE; they don't reduce rows.
- CTE referenced twice (or `AS MATERIALIZED`, or PG ≤11) = optimization fence.
- `ORDER BY x LIMIT 1` without index on x = full sort; hides until table exceeds memory.
- `NOT IN` + nullable subquery → zero rows; use `NOT EXISTS`.
- FK columns are not auto-indexed in Postgres — missing FK indexes slow joins and cascades.
- JSON columns are denormalization: wrong for anything you filter, join, or constrain.
- Denormalize only when: measured hot join + rarely-changing duplicate + written source of truth. Maintain precomputed aggregates in the same transaction or trigger, never "the app remembers."

## Verification / self-check

- Index recommendation: state the exact query shape; walk the prefix rule; confirm no function/cast wraps a column; name the existing index it makes redundant and drop it.
- Slow-query diagnosis: point at the divergent plan node; your fix must change that node, or you're guessing.
- Concurrency claim: name the anomaly, state the engine's actual default level, simulate two interleaved transactions statement by statement.
- Migration: state each statement's lock, what it blocks, for how long; confirm `lock_timeout` and deploy ordering.
- Capacity claim: rows × frequency × bytes vs buffer pool; connections × work_mem vs RAM — written down or unchecked.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 12 baseline (compressed to anchors), 1 partial (equality-column ordering by reusability vs "most selective first"), 0 delta.
- Opus cold reproduced: E-S-R with the LIMIT tiebreak, full lost-update/write-skew fix ladders, all DDL safety machinery incl. INVALID-index recovery and PG-version caveats, CTE fence versions, PgBouncer breakage list, visibility-map/Heap Fetches, expand/contract ordering.
- Kept expanded (unprobed rare-expert items): HOT-update disablement as the hidden write tax; idle-in-transaction as fleet-wide vacuum saboteur; the 5–10% OLAP boundary number.
