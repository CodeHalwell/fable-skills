---
name: caching-strategies
description: Load when adding or reviewing any cache — Redis/Memcached, CDN, HTTP caching, in-process memoization, or materialized views — or when debugging stale data, cache stampedes, low hit ratios, or invalidation bugs. Also load when someone proposes "just cache it" as a performance fix.
---

# Caching Strategies

Compact checklist. The standard machinery — cache-aside, delete-don't-set (and its residual read-old/fill-late race), single-flight locks, XFetch, serve-stale-while-revalidate, TTL jitter, negative-cache sentinels, key versioning/generation counters, replica-lag double-delete, fail-open-with-breaker, `Vary`/`private` hygiene — is assumed known; it appears here only as anchors so nothing gets skipped under pressure.

## Before caching anything

- Fix the query first: a 900ms query an index makes 5ms needs no cache, no invalidation, no stampede surface.
- Write down the maximum staleness the *business* tolerates (not what engineering finds convenient) and who is hurt when the bet loses. No answer = latent correctness bug with good latency.
- Never cache: uncommitted reads; anything used to *enforce* an invariant (balance/inventory gates — the source of truth decides); secrets in shared caches.
- Auth/permissions/entitlements: TTL ≤ 60s or don't cache — staleness there is a security incident.

## Invalidation strategy by data type (choose by failure mode, not hit ratio)

| Data | Strategy |
|---|---|
| Rarely changes, staleness cheap | TTL (minutes–hours) |
| Changes on write paths you own | Delete-on-write + TTL backstop |
| Written by many services/writers | CDC/event-driven invalidation + TTL backstop |
| Derived aggregates (counts, feeds) | Short TTL or periodic rebuild — never per-write invalidation of a hot aggregate |
| Writers you can't enumerate | TTL only — incomplete explicit invalidation is worse than none |

TTL fails predictably (bounded, self-healing); event-driven fails unpredictably (missed event = stale forever). Every explicit-invalidation scheme gets a TTL safety net. Batch jobs, admin tools, and second services are the write paths that skip your delete hook — that's what the backstop and CDC are for.

## Hot-key checklist (apply all)

1. TTL jitter (`base * uniform(0.9, 1.1)`) — keys populated together expire together.
2. Single-flight per key (in-process lock; cross-process `SET key:lock <token> NX EX` with delete-if-token-matches); losers serve stale.
3. Serve-stale: physical TTL ~2× logical expiry so stale data exists to serve during recompute and origin outages — but decide per key class; stale prices/permissions may be worse than down.
4. XFetch for the hottest keys.
5. **Read-rate hot keys (celebrity problem):** one key at 100k req/s lives on ONE Redis shard — adding shards does nothing. Fixes: in-process L1 with 1–5s TTL in front of Redis, or key replication (`key#{rand(0,9)}` read fanout, write/invalidate all copies). This is the piece most designs miss: sharding solves keyspace load, never single-key load.

## Key design

- Key = full function signature: entity, tenant, locale, currency, flag-state, schema version. Run the "second tenant / second locale / second flag-state" test.
- Schema version in the key from day one (`price:v7:{id}:{ccy}`): class-wide invalidation becomes a one-line version bump at deploy; without it you're running SCAN+UNLINK over 200M keys racing new bad writes.
- Per-entity generation counter for unbounded derived keys: `user:{id}:gen` embedded in dependents; invalidate-all = one INCR.
- Normalize multi-valued inputs (sort params) before keying; hash long keys, log the preimage.

## Pitfall one-liners (audit every one on review)

- Writer SETs cache instead of DELETE → old value can win the race and persist to TTL.
- Cache fills from a lagging read replica right after delete-on-write → fresh-looking stale fill (delayed double-delete, LSN-gated fill, or primary fills).
- Errors/timeouts cached through the success path → self-inflicted per-key outage.
- Negative cache using `null`-that-means-miss → no-op; sentinel must be distinct, short TTL, deleted on the create path.
- `Cache-Control: public` or missing `Vary` on authenticated responses → cross-user leak; `Vary: Cookie` → hit ratio silently ~0.
- Deploy flushes all in-process caches simultaneously → synchronized stampede on shared tier.
- Unbounded local dict / `functools.lru_cache` on instance methods (holds `self`, cache is per-class, no TTL) → memory leak; use `cachetools.TTLCache` or `cached_property`.
- Write-behind acknowledging writes it can lose → cache became the (non-durable) source of truth.
- No Redis client timeout/breaker → cache outage becomes hard-dependency outage; and verify the origin survives 0% hit ratio or pair fail-open with load shedding.
- Global 95% hit ratio hiding a 20% ratio on the expensive namespace → measure per key-class, weighted by origin cost saved; watch eviction rate and staleness-at-serve alongside.

## Verification / self-check

- Per cached value, one line each: max tolerable staleness? every write path? what deletes/expires on each path? what happens at 0% hit ratio? Any "unclear" = design not done.
- Interleave a writer and reader at every step: can an older value be written into the cache after a newer one?
- Every entry has a TTL; every cache has a size bound; hot keys have single-flight AND jitter; miss-heavy attacker-reachable lookups have negative caching.
- HTTP/CDN: read `Cache-Control` and `Vary` off a real response, not the config's intent.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 13 baseline (cut/compressed), 0 partial, 0 delta.
- Opus cold reproduced everything load-bearing: delete-vs-set race, stampede arithmetic + full mitigation stack incl. XFetch formula, version/generation invalidation, replica-lag double-delete, fail-open + origin-capacity question, negative-cache sentinels, lru_cache leak, Vary/private hygiene.
- Skill retained as a compact review checklist; only marginal expansions kept: celebrity-key single-shard fanout (L1/key-replication) and the business-staleness framing.
