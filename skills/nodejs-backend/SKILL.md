---
name: nodejs-backend
description: Loads expert Node.js production judgment for event-loop behavior, streams and backpressure, worker threads vs cluster, memory-leak hunting, unhandled rejection policy, ESM migration, framework selection (express/fastify/hono), graceful shutdown, and npm supply-chain hygiene. Use when building, reviewing, or debugging Node backend services and their deployment behavior.
---

# Node.js Backend

## Core mental model (anchors)

1. One thread runs your JavaScript; `await` yields but one synchronous 200ms `JSON.parse` delays every in-flight request. libuv pool (default 4) services fs/`dns.lookup`/zlib/pbkdf2-class crypto; sockets don't touch it.
2. Every producer-faster-than-consumer path needs named backpressure — `stream.pipeline()` or explicit drain handling; nothing else counts.
3. Rejections/errors have three escape routes (uncaughtException, unhandledRejection, unhandled `'error'` events); production declares a policy for all three — and the policy is crash-and-restart, not swallow.
4. `npm install` is an attack surface (Shai-Hulud era, Sept 2025 onward); process boundaries are the unit of resilience.

## Current state (verified July 2026)

- **Node 24 Active LTS**; 22 Maintenance; 26 Current (LTS Oct 2026); cadence moves to one major/year starting Node 27. Permission model stable on 24 (`node --permission --allow-fs-read=/app --allow-net`) — use it for services running risky dependency trees. `require(esm)` on all supported lines (22+, minus top-level-await graphs) ended the dual-package era; new services start `"type": "module"`, new libraries can ship ESM-only.
- `node:test`, `--watch`, `import.meta.dirname`, `--env-file` are stable — nodemon/dotenv/jest are now optional deps, not defaults.
- Framework default: **fastify** (schema-driven validation *and serialization* — the serializer is the underrated real-world win); **hono** when multi-runtime/edge portability or end-to-end typing dominates; express is legacy/middleware-museum. Don't propose framework rewrites unless HTTP overhead actually shows in profiles.

## Judgment calls and sharpened numbers

- **Loop-delay threshold:** instrument `monitorEventLoopDelay()` always-on; investigate when p99 exceeds ~20ms (not 100ms — by 100ms users already feel it). Fix order for blocking work: eliminate (cache, cap payload sizes, paginate) → stream → chunk with `setImmediate` → worker pool. Reaching for workers before elimination is the classic over-engineering move.
- **Scaling across cores, 2026:** run N single-process containers behind the orchestrator; `cluster`/pm2 only where no orchestrator exists. Externalize shared in-memory state (rate limiter, dedup set) to Redis *before* scaling out, not after the incident.
- **ESM migration stalls on test-time monkey-patching, not syntax** — ESM namespace bindings are immutable, so `sinon.stub(fs, 'readFile')`-style patching dies; budget for DI or vitest/`node:test` module mocking. Migrate existing CJS only under a forcing function.
- **`NODE_ENV=production` is still a real 2x on express apps** (view caching, library fast paths). Fastify doesn't care; your dependencies might.

## Diagnosis priors (compressed)

- p99-without-p50 movement = queueing behind someone else's synchronous work; APM attributes CPU to the request that burned it, so victims look slow and the culprit looks fine. Loop-delay first, `--cpu-prof`/0x second; suspect serialization of oversized payloads (the 40MB admin endpoint) before anything exotic. Fix is usually "don't build the payload" (paginate/stream NDJSON) — moving 40MB serialization to a worker is admitting the response is wrong-sized.
- Memory leak: heap-vs-native split first (`heapUsed` flat + RSS growing → Buffers/`external`/fragmentation; snapshots won't show it). Snapshot caveats: stop-the-world + needs ~heap-sized RAM headroom. Base rates: unbounded Map keyed by high-cardinality input (closure variant nastiest) > listener accumulation (grep logs for `MaxListenersExceededWarning` before anything fancy; per-request listeners on a shared `AbortSignal`) > in-flight registries whose delete is happy-path-only > shipped debug arrays. Nightly restarts mask it; leaks track load, so a spike still OOMs mid-day.

## Pitfalls checklist

Baseline one-liners (kept for completeness): `pipeline` not `.on('data')`/bare `.pipe`; `await once(dest, 'drain')` in write loops; graceful shutdown = readiness-flip → `server.close()` + `closeIdleConnections()` → deadline `setTimeout(...,10s).unref()` → close pools → exit 0; exec-form `CMD ["node","server.js"]` or `--init` (npm as PID 1 drops SIGTERM); `Promise.allSettled` or wired AbortSignal for side-effecting fan-out; recursive `setTimeout` over overlapping `setInterval`; late-binding rejections (`const p = doA(); await doB(); await p`) crash processes that "have try/catch everywhere"; `--max-old-space-size` ≈ 75–80% of container limit; bounded concurrency (`p-limit`) — unbounded `Promise.all` fan-out is a bug; body-size limits at framework/proxy; `keepAliveTimeout` (and `headersTimeout` above it) longer than LB idle timeout or intermittent 502s; `dns.lookup` is threadpool — keep-alive agents + `UV_THREADPOOL_SIZE` for fs+crypto+dns-heavy services.

Expanded — the non-obvious ones:

- **`p.catch(() => {})` marks a promise handled while a later `await p` still throws** — legitimate for deliberate deferred-await patterns, but comment it; it silently defuses the crash-on-unhandled policy for that promise.
- **Compose per-request abort with `AbortSignal.any([reqSignal, shutdownSignal])`** instead of adding per-request listeners to one long-lived signal (a leak that presents as slow listener growth).
- **Supply chain, post-worm specifics** (cold answers know the categories; these are the load-bearing settings): `.npmrc` `ignore-scripts=true` + allowlist via `@lavamoat/allow-scripts`; **cooldown ≥7 days** on dependency bumps (pnpm/Renovate `minimumReleaseAge` — the 2025–26 compromises were detected within days); review lockfile diffs like code — a bump that edits resolved URLs or adds install scripts is stop-the-line; `npm audit` is noise, behavior scanners (socket.dev-class) catch what CVE feeds miss; never `npx <unvetted>` in CI with credentials in env.

## Micro-example: the one worth keeping verbatim

Backpressure-correct streaming endpoint — slow client pauses gzip pauses the DB cursor:

```ts
await pipeline(
  db.streamRows('SELECT ...'),
  async function* (src) { for await (const r of src) yield JSON.stringify(r) + '\n'; },
  createGzip(),
  reply.raw,
);
```

## Verification and stopping rule

1. Every stream path: name its backpressure mechanism, or it's a memory bomb under a slow consumer.
2. Every promise has an owner, including on error branches and deferred-await patterns.
3. Shutdown tested under load in CI: SIGTERM, assert zero dropped responses; timers `.unref()`ed.
4. Performance claims need a flamegraph or loop-delay number; version-gate (permission model → 24; `require(esm)`/`import.meta.dirname` → 22+).
5. Dependency advice: check publish recency; scripts-disabled installs.

Done when loop delay is instrumented, rejections crash loudly, shutdown is load-tested, and installs run scripts-disabled + lockfile + cooldown.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 hard delta.
- Opus cold nails: Node 24 LTS + permission model + require(esm) with TLA limit, fastify/hono differentiators, loop-queueing p99 mechanism, leak-hunt workflow incl. snapshot caveats, pipeline/drain, full K8s shutdown incl. closeIdleConnections and PID 1, keepAliveTimeout relation, threadpool facts, supply-chain cooldown. This skill was ~95% baseline — the heaviest restructure of the batch.
- Remaining value: the 20ms loop-delay action threshold, `p.catch(()=>{})`-marks-handled nuance, AbortSignal.any composition, NODE_ENV-on-express 2x, allow-scripts/lockfile-diff specifics, "don't build the payload" over worker-offload judgment.
