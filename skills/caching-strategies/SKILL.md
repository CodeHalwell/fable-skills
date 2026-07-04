---
name: caching-strategies
description: Load when adding or reviewing any cache — Redis/Memcached, CDN, HTTP caching, in-process memoization, or materialized views — or when debugging stale data, cache stampedes, low hit ratios, or invalidation bugs. Also load when someone proposes "just cache it" as a performance fix.
---

# Caching Strategies

## Core mental model

- **A cache is a bet that stale data won't hurt.** Every cache entry is a claim: "the source can't have changed in a way that matters within this window." Before caching anything, write down the maximum staleness the *business* tolerates (not what engineering finds convenient) and who gets hurt when the bet loses. If nobody can answer, the cache is a latent correctness bug with good latency.
- **Invalidation strategies are ranked by their failure mode, not their hit ratio.** TTL fails *predictably* — bounded staleness, self-healing. Event-driven invalidation fails *unpredictably* — a missed event means stale forever. Prefer the strategy whose failure you can tolerate, and back every explicit-invalidation scheme with a TTL safety net so the worst case is bounded.
- **Caches hide load until they don't.** A 99% hit ratio means the origin sees 1% of traffic — and gets provisioned for that. A cold restart, a mass expiry, or a key-version bump sends 100× to an origin that has never seen it. Capacity-plan the origin for realistic miss storms, or make warm-up part of deploys.
- **Cache-aside is the default topology; everything else needs a reason.** App reads cache → on miss, reads source → populates cache with TTL. Write path updates the source and *deletes* (not updates) the cache key. Write-through and write-behind couple the cache into the write path and add failure modes; adopt them only for the specific problems they solve (read-after-write locality; write buffering with explicitly accepted data-loss risk).
- **A cache key is a function signature.** The key must encode *every* input that affects the value: entity ID, tenant, locale, currency, feature-flag state, serialization/schema version. Any input missing from the key is a cross-contamination bug waiting for the second variant of that input to show up.

## Decision frameworks

### Invalidation strategy by data type

| Data | Strategy | Reasoning |
|---|---|---|
| Rarely changes, staleness cheap (config, product descriptions) | TTL (minutes–hours) | Simplest; failure mode is bounded, self-healing staleness |
| Changes on known write paths you own (user profile) | Delete-on-write + TTL backstop (hours) | Precise freshness; TTL caps the damage of a missed delete |
| Changes from many writers / other services | Event-driven (CDC or bus) invalidation + TTL backstop | You can't hook every writer; CDC catches them all; TTL catches CDC gaps |
| Derived aggregates (counts, feeds) | Short TTL or periodic rebuild; never per-write invalidation | Per-write invalidation of a hot aggregate = one key deleted constantly = stampede machine |
| Auth/permissions/entitlements | TTL ≤ 60s, or don't cache | Staleness here is a security incident, not a UX blemish |
| Anything whose writers you can't enumerate | TTL only | Explicit invalidation you can't make complete is worse than none — unbounded staleness with false confidence |

Delete, don't set, on invalidation: writing the new value into the cache from the write path races with concurrent cache-aside fills and can leave the *older* value winning (see pitfalls). Deleting is idempotent and race-tolerant — the next read repopulates from the source.

### What's safe to cache

- **Safe:** idempotent reads keyed by all their inputs; immutable or content-addressed data (cache forever — the ideal cache); public data at the CDN.
- **Dangerous:** anything personalized at a shared layer (CDN/proxy) — one missing `Vary` or key component and user A sees user B's account page; permission-derived data; paginated lists whose pages shift under writes (cache page 1 only, or key by cursor, not page number).
- **Never:** uncommitted or transactional reads; anything used to *enforce* an invariant (balance checks, inventory gates — the source of truth must make that decision); secrets and tokens in shared caches without encryption and short TTLs.

### Cache stampede (dog-pile) prevention — apply ALL of these for hot keys

1. **TTL jitter:** `ttl = base * random.uniform(0.9, 1.1)`. Keys populated together (deploy, warm-up script, midnight cron) otherwise expire together, and the misses arrive as a synchronized wave.
2. **Single-flight:** on miss, exactly one caller recomputes; concurrent callers wait or serve stale. In-process: Go's `singleflight`, or a per-key `asyncio.Lock`. Cross-process: `SET key:lock <token> NX EX 10`; losers poll briefly or serve the stale value; holder deletes only if the token still matches. Never let N processes all recompute the same key.
3. **Probabilistic early expiry (XFetch):** store `(value, delta, expiry)` where delta = the last recompute's duration; a reader refreshes early when `now - delta * beta * log(random()) >= expiry` (beta ≈ 1). Hot keys get refreshed before expiry by a randomly chosen reader; cold keys just expire. This removes the synchronized miss entirely for frequently-read keys.
4. **Serve-stale-while-revalidate:** keep entries physically past logical expiry; serve stale during recompute and during origin outages (`stale-if-error` semantics). Usually the right availability call — but decide it explicitly per key class, because for some data (prices, permissions) stale-on-error is worse than down.

### Negative caching

- Cache "not found" results with a short TTL (5–60s) whenever misses are expensive or attacker-reachable. Without it, every request for a nonexistent key passes straight through — the classic cache-penetration DoS: iterate random IDs, hit ratio 0%, origin dies.
- Use a distinct sentinel value (`"__MISSING__"`, or a wrapper object), never `null`-that-also-means-cache-miss — otherwise every negative hit re-queries the origin and the negative cache does nothing.
- Invalidate the negative entry on the creation path (user signs up → delete the "no such user" entry), or newly created entities appear broken for the TTL.
- For huge sparse keyspaces under attack, a Bloom filter of existing keys in front of the cache is the heavier-duty version.

### Layering — cache as close to the user as staleness allows

| Layer | Latency saved | Scope | Cache here when |
|---|---|---|---|
| CDN / edge | 10–300ms per request | Shared, global | Public or coarsely-varied content. Set `Cache-Control` deliberately: `s-maxage` for the CDN, `max-age` for browsers, `stale-while-revalidate` for smoothness. Personalize via edge-included fragments or client-side fetch — don't make the whole page uncacheable for one username |
| App cache (Redis/Memcached) | ~0.5–2ms lookups vs 10–500ms recompute | Shared across instances | The default layer for computed/DB-derived values; survives deploys; one copy of the truth-as-of-TTL |
| In-process (dict / LRU / Caffeine) | ns–µs | Per instance | Ultra-hot, small, short-TTL data (flags, schema metadata). N instances = N independent stale copies — keep TTL ≤ seconds, and remember invalidation can't reach into other processes |
| DB-side (buffer pool, materialized views) | — | Shared | The buffer pool is free caching: don't build Redis machinery to cache what the DB already serves from memory in 1ms. Materialized views for expensive aggregates with scheduled refresh |

Rule: fix the query before caching it. Caching a 900ms query that an index makes 5ms adds staleness, a stampede surface, and an infra dependency to a problem with a one-line fix.

### Cache-key design and namespace versioning

- Canonical, deterministic keys: `{app}:{entity}:{schema-version}:{id}:{variant}`, e.g. `shop:product:v3:12345:en-GB`. Normalize multi-valued parts (sort query params) before hashing — `?a=1&b=2` and `?b=2&a=1` must produce one key.
- **Version-bump instead of scanning:** to invalidate a whole class (schema change, bug in cached values), bump `v3` → `v4` in code; old entries die by TTL/LRU. Never `KEYS pattern` in production Redis (it blocks the event loop), and avoid `SCAN`+delete sweeps (slow, racy).
- Per-entity generation counters for "invalidate everything about user X": store `user:12345:gen = 7`; embed it in dependent keys (`feed:12345:g7`). Invalidation = one `INCR`; unbounded derived keys die at once. Cost: one extra cache read per lookup — pipeline it with the main read.
- Hash long keys (e.g., SHA-1 of the canonical string) to keep them short, but log the preimage for debuggability.

## Failure modes & pitfalls

- **Set-on-write race (the classic).** Writer updates DB then SETs the cache; meanwhile a reader missed, read the *old* DB value, and SETs it after the writer → stale value persists until TTL. Correction: writers DELETE, readers fill. (Cache-aside alone has a tiny version of the same race — read-old → concurrent delete → fill-old — which is why the TTL backstop is non-negotiable even with delete-on-write.)
- **Caching before fixing the bug/query.** "It's slow, add Redis" — now it's slow *sometimes* and wrong *sometimes*. Profile first. Cache is the tool for load you can't remove, not the first tool.
- **Personalized data cached at a shared layer.** `Cache-Control: public` (or a proxy default) on an authenticated response → user B receives user A's page; this exact bug has caused real-world account-data leaks. Set `Cache-Control: private, no-store` on session-derived responses, and audit `Vary`: missing `Vary` on an auth-relevant dimension = cross-user leakage; `Vary: Cookie` = hit ratio silently ~0 because every user is a cache miss.
- **Thundering herd on deploy.** Deploy flushes in-process caches on every instance simultaneously; all instances stampede Redis/DB at once. Corrections: never flush shared caches on deploy; warm critical keys as a deploy step; rely on jittered lazy fills for the rest.
- **Hot-key overload.** One celebrity key gets 100k req/s to a single Redis shard — adding shards doesn't help; it's ONE key on ONE shard. Corrections: in-process L1 with 1–5s TTL in front of Redis for the hottest keys; or key replication (`key#{rand(0,9)}` read fanout, all-copies write/invalidate).
- **Missed invalidation via the "other" write path.** Cache deleted in the API code path, but a batch job, admin tool, or second service writes the same rows directly → stale forever if there's no TTL. Correction: TTL on everything, no exceptions; or invalidate from CDC (Debezium → bus → delete) so *any* writer triggers it regardless of code path.
- **Unbounded local caches = memory leak.** A module-level dict keyed by user ID grows forever. `functools.lru_cache` on an instance method holds `self` alive (leaks every instance) and has no TTL. Use `cachetools.TTLCache(maxsize=..., ttl=...)` or a Caffeine-style bounded cache. Every cache needs a max size AND an eviction story, stated.
- **Caching errors/timeouts as values.** Origin times out; `None` or an exception placeholder gets cached for the full TTL → a self-inflicted 5-minute outage per key. Cache failures only under an explicit short negative-TTL policy, never through the success path's code.
- **Cache as accidental source of truth.** Write-behind cache acknowledges writes, crashes before flushing → acknowledged data gone. If durability matters, the DB write is synchronous, full stop; write-behind is only for data you can afford to lose and re-derive.
- **Redis-down = site-down.** A cache client with no timeout or breaker turns a cache outage into a hard dependency outage. Corrections: aggressive client timeouts (50–100ms), treat cache errors as misses, circuit-break to stop hammering a dead cache — and verify the origin actually survives a 0% hit ratio, or pair the fallback with load-shedding.
- **TTL as a race with replication lag.** Cache filled from a read replica that's 5s behind, right after a delete-on-write → the "fresh" fill is stale data with a full TTL. Fill from the primary after invalidation-driven misses, or delay/re-issue the delete (delayed double-delete), or accept and document the window.
- **Measuring hit ratio globally and celebrating.** A 95% global ratio can hide a 20% ratio on the expensive key class, drowned out by cheap hot keys. Measure per key-namespace, and weight by *origin cost saved*, not request count. Watch alongside it: origin QPS during expiry windows (stampede tell), eviction rate (evictions with low hit ratio = undersized or key-churned), and staleness age at serve time. Near-100% with long TTLs may just mean you're serving very stale data; <50% on a namespace usually means the key embeds a too-unique component (timestamp, request ID) and that cache is pure overhead.

## Worked micro-examples

### 1. Single-flight + jitter + serve-stale in Python (asyncio)

```python
import asyncio, random, time

_locks: dict[str, asyncio.Lock] = {}

async def get_cached(key: str, recompute, ttl: float = 300.0):
    entry = await redis.hgetall(key)              # {value, expires_at}
    now = time.time()
    if entry and now < float(entry["expires_at"]):
        return entry["value"]                      # fresh hit
    lock = _locks.setdefault(key, asyncio.Lock())
    if lock.locked() and entry:                    # someone is already recomputing
        return entry["value"]                      # serve stale, don't pile on
    async with lock:                               # single flight per process
        entry = await redis.hgetall(key)           # re-check: loser of the lock race
        if entry and now < float(entry["expires_at"]):
            return entry["value"]
        fresh = await recompute()
        expires = now + ttl * random.uniform(0.9, 1.1)   # jitter
        await redis.hset(key, mapping={"value": fresh, "expires_at": expires})
        await redis.expire(key, int(ttl * 2))      # physical TTL > logical: stale stays servable
        return fresh
```

Load-bearing details: logical expiry stored *inside* the value with the physical TTL at 2× (so stale data still exists to serve); jitter on the logical TTL; re-check after acquiring the lock (the double-checked-locking of caching — without it every lock waiter recomputes serially); lock holders recompute while waiters serve stale. For cross-process single-flight, replace the local lock with `SET key:lock <token> NX EX 10` and delete-if-token-matches on release.

### 2. Stampede arithmetic (why single-flight is survival, not optimization)

Key: rendered homepage module. Recompute = 800ms of DB time. Traffic = 2,000 req/s. TTL expires with no protection: during the 800ms recompute window, 1,600 requests miss and *all* recompute → 1,600 concurrent 800ms queries ≈ 1,280 seconds of DB work demanded instantly. The DB has maybe 32 usable cores/connections → the queue explodes, recomputes slow to many seconds, more requests miss, and the system enters metastable collapse — from one key. With single-flight: exactly 1 recompute and 1,599 stale serves, origin load unchanged.

### 3. Key-versioning invalidation in one line

Bug ships that caches prices with the wrong currency conversion. Wrong fix: emergency script to SCAN and delete `price:*` across a 200M-key Redis (hours, and racing new bad writes). Right fix: the key template was `price:v7:{product}:{currency}`; deploy with `v7` → `v8`. Invalidation is complete at deploy time, instant, and needs no cache round-trips; the `v7` corpse expires by TTL/LRU. This only works if the version was in the key from day one — which is why it always should be.

## Verification / self-check

- For every cached value, answer in one line each: maximum tolerable staleness? every write path that changes the source? what deletes/expires the entry on each path? what happens at 0% hit ratio? If any answer is "unclear", the design isn't done.
- Check the key against the value's full input list — run the "second tenant / second locale / second flag-state" test mentally.
- Interleave a writer and a reader at every step: can an old value be written into the cache after a newer one? (If the write path SETs the cache, the answer is yes.)
- Confirm every entry has a TTL, every cache has a size bound, and hot keys have single-flight AND jitter.
- Confirm negative caching exists on miss-heavy or attacker-reachable lookups, with a sentinel distinct from "miss".
- For HTTP/CDN: read the actual `Cache-Control` and `Vary` headers off a real response — not the config's intent — and check an authenticated response is `private`.
