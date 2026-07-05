---
name: nodejs-backend
description: Loads expert Node.js production judgment for event-loop behavior, streams/backpressure, worker threads vs cluster, memory-leak hunting, unhandled rejections, ESM migration, framework selection (express/fastify/hono), graceful shutdown, and npm supply-chain hygiene. Use when building, reviewing, or debugging Node backend services and their deployment behavior.
---

# Node.js Backend

## Core mental model

1. **One thread runs your JavaScript; everything else is a lie of concurrency.** The event loop interleaves callbacks on a single thread; libuv's thread pool (default size 4 — `UV_THREADPOOL_SIZE`) handles fs, dns.lookup, zlib, and crypto.pbkdf2-style work; sockets use OS async I/O directly. Consequence: `await` yields, but a synchronous 200ms JSON.parse blocks *every* request. "Node is slow" almost always means "something synchronous is on the loop."
2. **Microtasks starve macrotasks.** Order per loop turn: run a macrotask (timer/IO callback) → drain the *entire* microtask queue (promises, `queueMicrotask`) plus `process.nextTick` (which runs even before other microtasks) → next macrotask. A promise chain that keeps scheduling microtasks (or recursive `nextTick`) starves I/O completely; `setImmediate` is the correct "yield to the loop" primitive for chunked CPU work.
3. **Backpressure is the difference between a stream and a memory bomb.** Every producer-faster-than-consumer path (file→HTTP, DB→transform→S3) must propagate "slow down." `pipeline()` does this for you; hand-rolled `.on('data')` → `.write()` loops do not.
4. **Rejections and errors have different escape routes.** Sync throw in a callback → `uncaughtException`; rejected promise nobody awaits → `unhandledRejection`; error on a stream/EventEmitter with no `'error'` listener → crash. A production service needs an explicit policy for all three, not defaults.
5. **The npm install step is an attack surface, not a formality.** Post-2025 (Shai-Hulud worm and successors), treat every dependency update as untrusted code execution at install time and runtime.

## Current state (verified July 2026)

- **Node 24 is Active LTS** (Node 22 in Maintenance; Node 26 is Current, LTS in Oct 2026; cadence moves to one major/year from Node 27). On 24: the **permission model is stable** (`--permission` with `--allow-fs-read`, `--allow-net`, etc. — use it to sandbox risky services), `URLPattern` is global, V8 13.6.
- **`require(esm)` works** across all supported lines (22+): CommonJS can load ES modules synchronously (no top-level await in the loaded graph). This ended the dual-package era — new libraries can ship ESM-only; new services should be `"type": "module"` from day one.
- **Framework landscape 2026:** Express still has the mindshare and middleware museum but is architecturally 2010s (callback middleware, slow JSON path, weak TS). **Fastify** is the default choice for serious Node-only HTTP services: schema-based validation/serialization (JSON Schema; pairs with TypeBox/zod), plugin encapsulation, 2–3x express throughput. **Hono** is Web-Standards (Request/Response) based, runs on Node/Bun/Deno/Workers/Lambda, best-in-class TS inference; pick it for edge portability or lightweight APIs. NestJS remains the "we want Spring" option for large teams wanting imposed structure. Recommend: fastify for a Node production API by default; hono if edge/multi-runtime or RPC-style type sharing with a frontend matters; express only for legacy or where its middleware ecosystem is decisive.

## Decision frameworks with reasoning chains

**Worker threads vs cluster vs child processes.** First question: *what am I actually parallelizing?* (1) CPU-bound work inside a request path (image resize, crypto, parsing huge payloads) → `worker_threads` via a **pool** (e.g., piscina) — threads share memory (`ArrayBuffer` transfer/`SharedArrayBuffer`), spawn cost is too high per-task, so never spawn per request. (2) Scaling an entire HTTP server across cores → in 2026 the honest answer is usually *neither cluster nor workers*: run N containers/processes behind your orchestrator's load balancing; `cluster` (or pm2) is for bare-metal/VM deployments without an orchestrator. (3) Running foreign binaries, untrusted code, or things that crash → `child_process`/`execa` — process isolation is the point. What changes the answer: shared in-memory state (rate limiters, caches) breaks under both cluster and multi-container — move it to Redis before scaling out, not after the bug report.
**Blocking-work triage:** measure first with `perf_hooks.monitorEventLoopDelay()` or clinic.js/0x flamegraphs. If p99 loop delay > ~20ms, find the sync frame (common culprits: `JSON.stringify` of megabyte objects, sync zlib, `bcrypt.hashSync`, regex catastrophic backtracking, huge `Array.sort`). Fix order: make it async/streaming → chunk it with `setImmediate` → move to worker pool. Don't reach for workers before confirming the work can't just be eliminated (cache the stringify, cap payload sizes).

**Unhandled rejection policy.** Node's default since v15 is crash on unhandled rejection — keep it. The decision: `process.on('unhandledRejection', ...)` handler should log with context and re-throw/exit(1), letting the supervisor restart. Never install a swallow-and-continue handler: after an unexpected rejection you cannot prove invariants (connections half-open, locks held). Same for `uncaughtException` — the docs-sanctioned use is synchronous cleanup + exit, nothing else. The subtle bug to hunt instead: *rejections that become unhandled later* — `const p = doThing(); await other(); await p;` — if `other` throws, `p`'s rejection is unhandled. Attach handlers at creation (`Promise.allSettled`, or `p.catch(noop)` markers) when you intentionally defer awaiting.

**ESM migration realities.** New code: `"type": "module"`, done. Existing CJS codebase: migrate only with a forcing function (an ESM-only dependency you can't `require(esm)` due to top-level await, or tooling). The real costs: `__dirname`/`__filename` gone (use `import.meta.dirname`/`import.meta.filename`, stable in modern lines); `require.cache` deletion tricks for hot reload don't exist in ESM; jest/ts-jest friction (vitest handles ESM natively — often the pragmatic unlock); deep-import paths break against `"exports"` maps; and **monkey-patching for tests dies** — ESM namespace objects are immutable bindings, so `sinon.stub(fs, ...)`-style patching of imports fails; use dependency injection or `node:test`'s module mocking instead.

## How an expert thinks through it: a slow memory leak

RSS climbs 30MB/day; container OOMs weekly. Internal monologue: *First, is it heap or native? Compare `process.memoryUsage()`: if `heapUsed` is flat but RSS grows — native/Buffer/fragmentation territory (check `external`, `arrayBuffers`) — different hunt. Say heapUsed grows. Get three heap snapshots: baseline after warmup, after 1h, after 3h (via `node --inspect` + Chrome DevTools, or `v8.writeHeapSnapshot()` on SIGUSR2 — beware: snapshot pauses the process and needs ~heap-size RAM headroom). In DevTools, use Comparison view sorted by size delta, and read Retainers on the biggest growing class. Priors, in order of base rate: (1) unbounded in-process cache/Map keyed by something high-cardinality (user ID, URL) — the closure-in-cache variant is nasty because each entry retains its whole closure scope; (2) EventEmitter listeners added per-request and never removed — usually announced by the MaxListenersExceededWarning in logs, grep for that first, it's free; (3) closures capturing large request bodies in long-lived structures (a promise queue, an in-flight registry that never deletes on error paths); (4) module-level arrays used for "debugging" that ship to prod. Suppose retainers show a `Map` in `metrics.js` keyed by raw URL — including querystrings — so cardinality is unbounded. Fix: key by route pattern, cap with an LRU (`lru-cache` with `max` and `ttl`). Rejected alternatives: periodic process restart (masks it; leaks that follow load will still OOM during spikes); `global.gc()` calls (GC isn't the problem — reachability is); bumping `--max-old-space-size` (buys days, changes nothing).* Verify: replay load 24h, heapUsed sawtooths flat.

## Failure modes and pitfalls

- **Hand-rolled stream plumbing without backpressure.** `src.on('data', c => dest.write(c))` ignores `write()`'s `false` return — memory balloons at consumer stalls. Use `await pipeline(src, transform, dest)` (`node:stream/promises`) — it propagates backpressure AND errors AND calls destroy on all parts. `src.pipe(dest)` alone is a lesser trap: backpressure yes, but errors don't propagate and streams leak on failure; `pipeline` replaced it for a reason.
- **Async iteration over streams that write:** `for await (const chunk of src) { dest.write(chunk) }` re-loses backpressure — inside such loops you must `if (!dest.write(chunk)) await once(dest, 'drain')`, or just use `pipeline` with an async generator transform.
- **The listener-per-request leak.** `req.on('close', ...)` on a *shared* object (db client, global emitter) instead of the request object; or `AbortSignal` listeners: `signal.addEventListener('abort', cb)` on a long-lived signal from every request retains every `cb`. Remove listeners on completion, or use `{ once: true }` + `AbortSignal.any` composition.
- **`dns.lookup` surprise:** it's the libuv *threadpool*, not async DNS — under DNS slowness, 4 stuck lookups freeze fs/zlib/crypto too. Bump `UV_THREADPOOL_SIZE` for services doing crypto+fs+dns concurrently, and set keepAlive agents so you resolve less (`undici` pools do this well; `fetch` in Node is undici).
- **Graceful shutdown done wrong** (most common: not done). Correct sequence on SIGTERM: (1) stop accepting: `server.close()` — but note it waits for in-flight requests yet by default keep-alive idle sockets can hang it: call `server.closeIdleConnections()` (and after a grace period `closeAllConnections()`); (2) flip readiness probe to failing so the LB drains; (3) finish in-flight work with a deadline (`setTimeout(force, 10_000).unref()`); (4) close pools (DB, Redis) *after* traffic stops; (5) `process.exit(0)`. Also: any `setInterval` or open handle without `.unref()` prevents natural exit — that's why "my process won't die" happens. Test shutdown in CI: send SIGTERM under load, assert zero dropped responses.
- **Docker PID 1 problem:** `CMD node server.js` (exec form) is fine, but `CMD npm start` means npm is PID 1 and *does not forward SIGTERM* reliably — your graceful shutdown never runs. Run node directly, or use tini/`--init`.
- **`Promise.all` on writes:** one rejection returns control while sibling operations continue un-awaited — partial-write states plus possible late unhandled rejections. For side-effecting fan-out use `Promise.allSettled` and inspect, or an AbortSignal-connected cancellation.
- **npm hygiene (post-Shai-Hulud era, non-negotiable):** commit lockfiles and install with `npm ci`; set `ignore-scripts=true` in `.npmrc` (install scripts are the #1 initial-execution vector; allowlist the few packages that truly need builds, e.g. via `@lavamoat/allow-scripts`); enforce a **cooldown/quarantine window** — don't auto-merge dependency bumps published < 7 days ago (registry-poisoning campaigns are typically caught within days; pnpm has `minimumReleaseAge`, Renovate has `minimumReleaseAge`); pin exact versions for direct deps in applications; require 2FA/trusted-publishing for anything your org publishes; `npm audit` is noise-prone — signature/behavior scanners (socket.dev-class tooling) catch what CVE feeds don't. Never run `npx <unvetted>` in CI with credentials in env.
- **Config drift between dev/prod loop behavior:** `NODE_ENV=production` changes express (view cache, error verbosity) and many libs' perf paths; forgetting it is still a real 2x on express apps.

## Worked micro-examples

**Backpressure-correct streaming endpoint (fastify):**
```ts
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

app.get('/export', async (req, reply) => {
  const rows = db.streamRows('SELECT ...');          // Readable in object mode
  reply.raw.writeHead(200, { 'content-encoding': 'gzip', 'content-type': 'application/x-ndjson' });
  await pipeline(
    rows,
    async function* (src) { for await (const r of src) yield JSON.stringify(r) + '\n'; },
    createGzip(),
    reply.raw,                                        // client slowness pauses the DB cursor
  );
  return reply;                                       // tell fastify the response was handled
});
```

**Event-loop lag guardrail (cheap, always-on):**
```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  const p99ms = h.percentile(99) / 1e6;
  metrics.gauge('event_loop_p99_ms', p99ms);
  if (p99ms > 100) log.warn({ p99ms }, 'event loop blocked'); // then go find the sync frame with a flamegraph
  h.reset();
}, 10_000).unref();
```

## Verification and stopping rule

Before presenting Node backend advice or code: (1) walk every stream path and name the backpressure mechanism (pipeline? drain handling?) — if you can't name it, it's a leak; (2) walk every promise and confirm someone awaits or explicitly handles it, including in error branches; (3) confirm shutdown handles SIGTERM with connection draining and that node is PID 1; (4) for perf claims, require a flamegraph or loop-delay number, not vibes; (5) check version floors before recommending APIs (permission model stable on 24; `import.meta.dirname` and `require(esm)` on 22+; don't cite Node-26-only features for LTS deployments). Stop hardening when the loop-delay metric is instrumented, rejections crash loudly, shutdown is tested, and installs run with scripts disabled + lockfile + cooldown — beyond that, chasing framework micro-benchmarks (express→fastify→hono rewrites for their own sake) is waste unless HTTP overhead actually shows in your profiles.
