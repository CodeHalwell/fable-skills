---
name: caching-strategies
description: Load when adding or reviewing any cache — Redis/Memcached, CDN, HTTP caching, in-process memoization, or materialized views — or when debugging stale data, cache stampedes, low hit ratios, or invalidation bugs. Also load when someone proposes "just cache it" as a performance fix.
---

# Caching Strategies

## Core mental model

- **A cache is a bet that stale data won't hurt.** Every cache entry is a claim: "the source can't have changed in a way that matters within this window." Before caching anything, write down the maximum staleness the *business* tolerates (not what engineering finds convenient) and who gets hurt when the bet loses. If nobody can answer, the cache is a latent correctness bug with good latency.
- **Invalidation strategies are ranked by their failure mode, not their hit ratio.** TTL fails *predictably* (bounded staleness, self-healing). Event-driven invalidation fails *unpredictably* (a missed event = stale forever). Prefer the strategy whose failure you can tolerate, and back every explicit-invalidation scheme with a TTL safety net so the worst case is bounded.
- **Caches hide load until they don't.** A 99% hit ratio means the origin sees 1% of traffic — and is provisioned for that. A cold restart, a mass expiry, or a key-version bump sends 100× to an origin that has never seen it. Capacity-plan the origin for realistic cache-miss storms, or make cache warm-up part of deploys.
- **Cache-aside is the default topology; everything else needs a reason.** App reads cache → on miss, reads source → populates cache with TTL. Write path updates the source and *deletes* (not updates) the cache key. Write-through and write-behind couple the cache into the write path and add failure modes; adopt them only for the specific problems they solve (read-after-write locality; write buffering with acknowledged data-loss risk).
- **A cache key is a function signature.** The key must encode *every* input that affects the value: entity ID, tenant, locale, currency, feature-flag state, serialization/schema version. Any input missing from the key is a cross-contamination bug waiting for the second variant of that input.

## Decision frameworks

**Invalidation strategy by data type:**

| Data | Strategy | Reasoning |
|---|---|---|
| Rarely changes, staleness cheap (config, product descriptions) | TTL (minutes–hours) | Simplest; failure mode is bounded, self-healing staleness |
| Changes on known write paths you own (user profile) | Delete-on-write + TTL backstop (hours) | Precise freshness; TTL caps the damage of a missed delete |
| Changes from many writers / other services | Event-driven (CDC or bus) invalidation + TTL backstop | You can't hook every writer; CDC catches them all; TTL catches CDC gaps |
| Derived aggregates (counts, feeds) | Short TTL or periodic rebuild; never per-write invalidation | Per-write invalidation of aggregates = one hot key deleted constantly = stampede machine |
| Auth/permissions/entitlements | TTL ≤ 60s, or don't cache | Staleness here is a security incident, not a UX blemish |
| Anything you can't enumerate the writers of | TTL only | Explicit invalidation you can't make complete is worse than none — it creates *unbounded* staleness with false confidence |

Delete, don't set, on invalidation: writing the new value into the cache from the write path races with concurrent cache-aside fills and can leave the *older* value winning (see pitfalls). Deleting is idempotent and race-tolerant (the next read repopulates).

**What's safe to cache:**
- Safe: idempotent reads keyed by all inputs; immutable or content-addressed data (cache forever); public data at the CDN.
- Dangerous: anything personalized at a shared layer (CDN/proxy) — one missing `Vary`/key-component and user A sees user B's account page; permission-derived data; paginated lists whose pages shift under writes (cache page 1 only, or key by cursor).
- Never: uncommitted/transactional reads; anything used to enforce an invariant (balance checks, inventory gates — the source must decide); secrets and tokens in shared caches without encryption and short TTLs.

**Stampede (dog-pile) prevention — apply ALL for hot keys:**
1. **TTL jitter:** `ttl = base * random.uniform(0.9, 1.1)` — otherwise keys populated together (deploy, warm-up script) expire together and the misses arrive as a wave.
2. **Single-flight:** on miss, one caller recomputes; concurrent callers wait or serve stale. In-process: Go `singleflight`, or a per-key `asyncio.Lock`. Cross-process: `SET key_lock token NX EX 10`; losers either poll briefly or serve the stale value. Never let N processes all recompute.
3. **Probabilistic early refresh (XFetch):** store `(value, delta, expiry)` where delta = last recompute cost; refresh early when `now - delta * beta * log(random()) >= expiry` (beta≈1). Hot keys refresh before expiry, spread across callers; cold keys just expire. This eliminates the synchronized miss entirely for frequently-read keys.
4. **Serve-stale-while-revalidate:** keep entries past logical expiry; serve stale during recompute and during origin outages (`stale-if-error`). Usually the right availability call — decide it explicitly, per key class.

**Negative caching:** cache "not found" results (short TTL, 5–60s) whenever misses are expensive or attackable — otherwise requests for nonexistent keys pass straight through to the origin every time (classic cache-penetration DoS: attacker iterates random IDs, hit ratio 0%, origin dies). Use a distinct sentinel value, never `null`-that-means-miss, or every negative hit re-queries the origin. Invalidate the negative entry on the creation path (user signs up → delete the "no such user" entry) or new entities appear broken for the TTL.

**Layering — cache as close to the user as staleness allows:**

| Layer | Latency | Scope | Cache here when |
|---|---|---|---|
| CDN / edge | ~10–50ms saved per request | Shared, global | Public or coarsely-varied content; set `Cache-Control` deliberately (`public, max-age`, `s-maxage`, `stale-while-revalidate`); personalize via edge-included fragments, not by making the page uncacheable |
| App cache (Redis/Memcached) | ~0.5–2ms | Shared across instances | Default layer for computed/DB-derived values; survives deploys |
| In-process (dict/LRU/Caffeine) | ~ns–µs | Per instance | Ultra-hot, small, very-short-TTL data (flags, schema metadata); N instances = N independent stale copies — keep TTL ≤ seconds |
| DB-side (buffer pool, materialized views) | — | Shared | The buffer pool is free caching — don't build Redis machinery to cache what the DB already serves from memory in 1ms; materialized views for expensive aggregates with scheduled refresh |

Rule: fix the query before caching it. Caching a 900ms query that an index makes 5ms just adds staleness, a stampede surface, and infra to a problem with a one-line fix.

**Cache-key design and namespace versioning:**
- Canonical, deterministic keys: `{app}:{entity}:{schema-version}:{id}:{variant}` e.g. `shop:product:v3:12345:en-GB`. Sort/normalize any multi-valued parts (query params) before hashing — `?a=1&b=2` and `?b=2&a=1` must be one key.
- **Version-bump instead of scanning:** to invalidate a whole class (schema change, bug in cached values), bump `v3`→`v4` in code. Old entries die by TTL/LRU. Never `KEYS pattern` or `SCAN`+delete in production Redis — `KEYS` blocks the event loop; scan-deletes are slow and racy.
- Per-entity generation for "invalidate everything about user X": store `user:12345:gen = 7`; embed the generation in dependent keys (`feed:12345:g7`); invalidation = `INCR` the generation. One O(1) write invalidates unbounded derived keys. Cost: one extra cache read per lookup (pipeline it).
- Hash long keys (SHA-1 of the canonical string) but log the preimage for debuggability.

## Failure modes & pitfalls

- **Set-on-write race (the classic).** Writer updates DB then SETs cache; meanwhile a reader missed, read the *old* DB value, and SETs it after the writer's SET → stale value persists until TTL. Correction: writers DELETE; or use compare-and-swap tokens. The same race exists in cache-aside alone (read-old → evict → fill-old) but the window is tiny; delete-on-write keeps it tiny instead of unbounded.
- **Caching before fixing the bug/query.** "It's slow, add Redis" — now it's slow *and* sometimes wrong. Profile first; cache is the tool for load you can't remove, not the first tool.
- **Personalized data cached at a shared layer.** `Cache-Control: public` (or a proxy default) on an authenticated response → user B receives user A's page. Set `Cache-Control: private, no-store` on anything session-derived, and audit `Vary` headers: missing `Vary: Accept-Encoding`/`Origin`/auth-dimension = cross-user leakage; `Vary: Cookie` = hit ratio silently ~0.
- **Thundering herd on deploy.** Deploy flushes in-process caches on every instance simultaneously; all instances stampede Redis/DB at once. Corrections: don't flush shared caches on deploy; warm critical keys as a deploy step; jittered lazy fills.
- **Hot-key overload.** One celebrity key gets 100k req/s to a single Redis shard — sharding doesn't help because it's ONE key. Corrections: in-process L1 with 1–5s TTL in front of Redis for the hottest keys; or key replication (`key#{rand(0,9)}` fanout with all-copies invalidation).
- **Missed invalidation via the "other" write path.** Cache deleted in the API code path, but a batch job / admin tool / second service writes the same rows directly → stale forever (no TTL backstop). Correction: TTL on everything, no exceptions; or invalidate from CDC (Debezium → bus → delete) so *any* writer triggers it.
- **Unbounded local caches = memory leak.** A module-level dict keyed by user ID grows forever; `functools.lru_cache` on a method holds `self` alive (leaks instances) and has no TTL. Use `cachetools.TTLCache(maxsize=..., ttl=...)` or Caffeine-style bounded caches. Every cache needs a max size AND an eviction story.
- **Caching errors/timeouts as values.** Origin times out, `None`/exception result gets cached for the full TTL → self-inflicted 5-minute outage. Cache failures only under an explicit short negative-TTL policy, never through the success path.
- **Cache as source of truth by accident.** Write-behind cache acknowledges writes, crashes before flushing → data gone. If durability matters, the DB write is synchronous, full stop.
- **Redis-down = site-down.** Cache client with no timeout/circuit breaker turns a cache outage into a hard dependency. Corrections: aggressive client timeouts (~50–100ms), treat cache errors as misses, breaker to stop hammering a dead cache — and verify the origin survives 0% hit ratio (or add load-shedding for that mode).
- **Measuring hit ratio globally and celebrating.** A 95% global ratio can hide a 20% ratio on the expensive key class (drowned out by cheap hot keys). Measure per key-namespace, and weight by *origin cost saved*, not request count. Also watch: origin QPS during expiry waves (stampede tell), eviction rate (evictions >0 with low hit ratio = undersized or key-churned cache), and staleness age at serve time. A hit ratio near 100% with long TTLs isn't automatically good — it may mean you're serving very stale data and could shorten TTLs for free; conversely <50% on a namespace usually means the key includes a too-unique component (timestamp, request ID) and the cache is pure overhead.

## Worked micro-examples

**1. Single-flight + serve-stale in Python (asyncio):**

```python
import asyncio, random, time

_locks: dict[str, asyncio.Lock] = {}

async def get_cached(key: str, recompute, ttl: float = 300.0):
    entry = await redis.hgetall(key)              # {value, expires_at}
    now = time.time()
    if entry and now < float(entry["expires_at"]):
        return entry["value"]                      # fresh hit
    lock = _locks.setdefault(key, asyncio.Lock())
    if lock.locked() and entry:                    # someone is recomputing
        return entry["value"]                      # serve stale, don't pile on
    async with lock:                               # single flight per process
        fresh = await recompute()
        expires = now + ttl * random.uniform(0.9, 1.1)   # jitter
        await redis.hset(key, mapping={"value": fresh, "expires_at": expires})
        await redis.expire(key, int(ttl * 2))      # physical TTL > logical: keeps stale servable
    return fresh
```

Load-bearing details: logical expiry stored *in* the value with physical TTL at 2× (so stale data exists to serve); jitter on the logical TTL; lock-holders recompute while waiters serve stale. For cross-process single-flight, replace the local lock with `SET key:lock <token> NX EX 10` and delete-if-token-matches on release.

**2. Stampede arithmetic (why this matters).** Key: rendered homepage module, recompute = 800ms of DB time, 2,000 req/s. TTL expires with no protection: during the 800ms recompute window, 1,600 requests miss and *all* recompute → 1,600 concurrent 800ms queries ≈ 1,280 seconds of DB work demanded instantly. The DB has maybe 32 usable cores/connections → queue explodes, recomputes slow further, misses pile up — metastable collapse from one key. With single-flight: exactly 1 recompute, 1,599 stale serves. This is why single-flight is not an optimization but a survival requirement for any hot key.

## Verification / self-check

- For every cached value, answer in one line each: max tolerable staleness? every write path that changes the source? what deletes/expires the entry on each? what happens at 0% hit ratio? If any answer is "unclear", the design isn't done.
- Check the key against the value's full input list — replay the "second tenant / second locale / second flag-state" test mentally.
- Simulate: writer and reader interleaved at every step — can an old value be written after a new one? (If the write path SETs the cache, the answer is yes.)
- Confirm every entry has a TTL and every cache has a size bound.
- Confirm hot keys have single-flight AND jitter; confirm negative caching exists on miss-heavy lookups.
- For HTTP/CDN: read the actual `Cache-Control`/`Vary` headers in the response, don't trust the config intent.
