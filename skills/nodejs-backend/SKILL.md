---
name: nodejs-backend
description: Loads expert Node.js production judgment for event-loop behavior, streams and backpressure, worker threads vs cluster, memory-leak hunting, unhandled rejection policy, ESM migration, framework selection (express/fastify/hono), graceful shutdown, and npm supply-chain hygiene. Use when building, reviewing, or debugging Node backend services and their deployment behavior.
---

# Node.js Backend

## Core mental model

1. **One thread runs your JavaScript; everything else is a lie of concurrency.** The event loop interleaves callbacks on a single thread. libuv's thread pool (default size 4, `UV_THREADPOOL_SIZE`) services fs, `dns.lookup`, zlib, and pbkdf2-class crypto; network sockets use OS async I/O directly and don't touch the pool. Consequence: `await` yields, but one synchronous 200ms `JSON.parse` delays *every* in-flight request. "Node is slow" almost always decodes to "something synchronous is on the loop."
2. **Microtasks starve macrotasks.** Per loop turn: run a macrotask (timer/I/O callback) → drain `process.nextTick` queue → drain the *entire* promise microtask queue → next macrotask. A chain that keeps scheduling microtasks (or recursive `nextTick`) starves I/O indefinitely; `setImmediate` is the correct "yield to the event loop" primitive when chunking CPU work. `setTimeout(fn, 0)` also works but clamps and is slower.
3. **Backpressure is the difference between a stream and a memory bomb.** Every producer-faster-than-consumer path (DB→HTTP response, file→transform→S3) must propagate "slow down." `stream.pipeline()` does this and error cleanup for you; hand-rolled `.on('data')` → `.write()` loops do neither.
4. **Rejections and errors have three different escape routes**, each needing an explicit policy: sync throws in callbacks → `uncaughtException`; promises nobody awaits → `unhandledRejection`; `'error'` events on streams/emitters with no listener → immediate crash. Production services declare behavior for all three; defaults are not a policy.
5. **`npm install` is an attack surface, not a formality.** Since the Shai-Hulud worm era (Sept 2025 onward: Shai-Hulud 2.0, the axios compromise, cross-registry campaigns), treat every dependency update as untrusted code execution — at install time (lifecycle scripts) and at runtime.
6. **Process boundaries are the unit of resilience.** Node has no isolation inside a process: a leaked listener, a corrupted global, an OOM affects everything. Design for cheap process death and restart (crash-only + orchestrator), and keep state out of the process.

## Current state (verified July 2026)

- **Node 24 is Active LTS**; Node 22 is Maintenance; Node 26 is Current (LTS October 2026). Release cadence moves to one major per year starting Node 27. Target 24 for new deployments; don't cite 26-only features for LTS users.
- On Node 24: the **permission model is stable** — `node --permission --allow-fs-read=/app --allow-net server.js` sandboxes fs/net/workers; use it for services running risky dependency trees. `URLPattern` is a global. Built-in `fetch`/`WebSocket` are undici-backed and production-ready.
- **`require(esm)` works on all supported lines (22+)**: CommonJS can synchronously load ES modules (unless the ESM graph uses top-level await). This ended the dual-package era — new libraries can ship ESM-only; new services should start `"type": "module"`.
- `node:test` (built-in runner), `node --watch`, `import.meta.dirname/filename`, and native `--env-file` are stable on modern lines — many dev-dependency staples (nodemon, dotenv, sometimes jest) are now optional.
- **Framework landscape 2026:** Express remains the mindshare/middleware museum but is architecturally 2010s (callback middleware, slow JSON path, weak TS). **Fastify** is the default for serious Node-only HTTP APIs — JSON-Schema-driven validation *and serialization* (the serializer is a large real-world win), plugin encapsulation, 2–3x express throughput. **Hono** is Web-Standard (Request/Response) based, runs on Node/Bun/Deno/Workers/Lambda with best-in-class TypeScript inference — pick for edge portability or type-shared RPC with a frontend. NestJS is the "we want Spring" choice for large teams wanting imposed structure. Default recommendation: fastify for a production Node API; hono when multi-runtime or end-to-end typing dominates; express for legacy or a decisive middleware dependency.

## Decision frameworks with reasoning chains

**Worker threads vs cluster vs child processes.** First question: *what exactly am I parallelizing?*
1. CPU-bound work inside a request path (image resize, crypto, parsing megabyte payloads) → **`worker_threads` behind a pool** (piscina is the standard). Threads share memory (transfer `ArrayBuffer`s, share `SharedArrayBuffer`s), but thread spawn is milliseconds — never spawn per request.
2. Scaling a whole HTTP server across cores → in 2026 the honest answer is usually **neither**: run N single-process containers behind the orchestrator's load balancing. `cluster`/pm2 is the bare-metal/VM answer where no orchestrator exists.
3. Foreign binaries, untrusted code, crash-prone native modules → **`child_process`** (or execa) — the point is fault isolation, which threads don't give.
4. What flips the answer: any shared in-memory state (rate limiter, session cache, dedup set) breaks silently under cluster AND multi-container. Externalize it (Redis) *before* scaling out, not after the incident.

**Blocking-work triage.** Measure first: `perf_hooks.monitorEventLoopDelay()` in production, flamegraphs (0x, clinic.js, `node --cpu-prof`) offline. If loop-delay p99 exceeds ~20ms, find the synchronous frame. Usual suspects in base-rate order: `JSON.stringify/parse` of large objects, sync zlib/crypto (`gzipSync`, `pbkdf2Sync`, bcrypt's sync path), catastrophic-backtracking regexes on user input, huge `Array.prototype.sort`, ORM hydration of thousands of rows. Fix order: eliminate (cache, cap payload sizes, paginate) → stream it → chunk with `setImmediate` → worker pool. Reaching for workers before trying elimination is the classic over-engineering move.

**Unhandled rejection policy.** Node's default since v15 is crash-on-unhandled-rejection — *keep it*. The `process.on('unhandledRejection')` handler you add should log context and exit(1) (let the supervisor restart); a swallow-and-continue handler leaves the process in an unprovable state (half-open connections, held locks). Same for `uncaughtException`: docs-sanctioned use is synchronous cleanup then exit, nothing more. The subtler bug to actually hunt: *late-binding rejections* — `const p = doA(); await doB(); await p;` — if `doB` throws, `p` may reject with no handler attached yet → crash from code that "has try/catch everywhere." When intentionally deferring an await, attach a placeholder: create with `Promise.allSettled`, or note the pattern `p.catch(() => {})` marks it handled while a later `await p` still throws — use deliberately, comment it.

**ESM migration realities.** New code: `"type": "module"`, done. Existing CJS: migrate only under a forcing function (an ESM-only dep you can't `require(esm)` because of top-level await; tooling requirements). Budget for the real costs:
- `__dirname`/`__filename` → `import.meta.dirname`/`import.meta.filename` (modern lines).
- `require.cache`-deletion hot-reload tricks don't exist in ESM; use `node --watch` or process restart.
- **Test-time monkey-patching dies**: ESM namespace bindings are immutable, so `sinon.stub(fs, 'readFile')`-on-the-import patterns fail. Move to dependency injection or `node:test`/vitest module mocking. This — not syntax — is where migrations stall.
- Jest's ESM support remains friction; vitest handles ESM natively and is often the pragmatic unlock.
- Deep imports (`pkg/lib/internal.js`) break against `"exports"` maps; fix imports, don't beg maintainers to remove the map.

## How an expert thinks through it: a slow memory leak

RSS climbs ~30MB/day; container OOMs weekly.

Internal monologue: *First split the space: heap or native? `process.memoryUsage()` — if `heapUsed` is flat while RSS grows, suspect native memory, `Buffer`s (`external`, `arrayBuffers` fields), or allocator fragmentation; different hunt (heapsnapshots won't show it). Say `heapUsed` grows. Take three snapshots — post-warmup baseline, +1h, +3h — via `node --inspect` + Chrome DevTools Memory panel, or `v8.writeHeapSnapshot()` wired to SIGUSR2 in prod (caveats: the snapshot stop-the-world pauses the process and needs roughly heap-sized RAM headroom — don't casually snapshot a 4GB-heap pod at peak). DevTools Comparison view, sort by size delta, read Retainers on the top growing constructor. Priors by base rate: (1) unbounded in-process Map/object cache keyed by high-cardinality input (user ID, URL, JWT) — the closure variant is nastiest because each entry retains its whole captured scope; (2) listeners accumulating on long-lived emitters — free clue: grep logs for MaxListenersExceededWarning before doing anything fancy; also `AbortSignal` listeners added per-request on a shared signal; (3) an in-flight registry (promise map, request tracker) whose delete lives on the happy path only — error paths leak entries; (4) module-level "debug" arrays that shipped. Snapshot says: a `Map` in metrics.js keyed by raw URL — querystrings included, unbounded cardinality. Fix: key by route pattern; cap with `lru-cache` (`max`, `ttl`). Rejected: nightly restarts (masks it; leaks track load, so a traffic spike still OOMs you mid-day); `global.gc()` (reachability, not GC laziness, is the problem); raising `--max-old-space-size` (buys days, changes nothing).*

Verification: 24h load replay; `heapUsed` must sawtooth around a flat mean, and the retained-size delta between late snapshots ≈ 0 for the fixed class.

## Second scenario: "every request is slow, but only sometimes"

p50 is 12ms; p99 is 900ms; the spikes correlate with nothing obvious in APM traces (time "disappears" between middleware spans).

Internal monologue: *Time that vanishes between spans in a single-threaded runtime is event-loop queueing — some other request's synchronous work is running while mine waits. APM tools attribute CPU to the request that* burned *it, not the ones that* queued *behind it, so the victim requests look mysteriously slow while the culprit looks fine. Get the right signal: `monitorEventLoopDelay()` — if loop-delay p99 tracks the latency spikes, confirmed. Now find the culprit frame: `node --cpu-prof` in staging under replayed traffic, or 0x flamegraph; look for wide synchronous frames. Suspects by base rate: `JSON.stringify` of a fat response (the classic: an admin endpoint serializing 50MB of rows), zlib compression of large bodies on the loop (compression middleware with no size cap), a catastrophic regex on user input, bcrypt/scrypt sync in an auth path. Flamegraph shows `stringifySafe` over a 40MB analytics payload. Fixes: (a) don't build the payload — paginate/stream NDJSON via pipeline; the structural fix; (b) fastify's schema-based serializer only helps proportionally — 40MB is still 40MB of sync work; rejected as THE fix; (c) move serialization to a worker — works, but shipping 40MB across a thread boundary to serialize it is admitting the response is wrong-sized. Take (a). Rejected earlier: scaling out replicas — queueing is per-process, so more replicas dilutes but doesn't fix, and the next big payload spikes again.*

Prior: **p99-without-p50 movement in Node is queueing behind someone else's synchronous work; instrument loop delay first, profile second, and suspect serialization of oversized payloads before anything exotic.**

## Failure modes and pitfalls

- **Hand-rolled stream plumbing.** `src.on('data', c => dest.write(c))` ignores `write()`'s `false` return — memory balloons whenever the consumer stalls. Use `await pipeline(src, ...transforms, dest)` from `node:stream/promises`: it propagates backpressure AND errors AND destroys all parts on failure. Bare `.pipe(dest)` is the lesser trap — backpressure yes, but errors don't propagate and streams leak on failure; `pipeline` exists because of it.
- **Async iteration that writes loses backpressure again:** inside `for await (const chunk of src) { dest.write(chunk); }` you must check `if (!dest.write(chunk)) await once(dest, 'drain')` — or skip the loop and give `pipeline` an async-generator transform.
- **Listener leaks per request.** Attaching handlers to *shared* objects (db client, global emitter, a long-lived `AbortSignal`) once per request retains every closure. Remove on completion, use `{ once: true }`, or compose per-request signals with `AbortSignal.any([reqSignal, shutdownSignal])`.
- **`dns.lookup` runs on the libuv threadpool** — not async DNS. Under slow DNS, four stuck lookups also freeze fs/zlib/crypto work. Mitigate: keep-alive agents (undici pools; Node's `fetch` is undici) so you resolve rarely, and/or raise `UV_THREADPOOL_SIZE` for fs+crypto+dns-heavy services.
- **Graceful shutdown done wrong** — or, most commonly, not at all. Correct SIGTERM sequence: (1) flip readiness to failing so the LB drains; (2) `server.close()` (stops new connections, waits for in-flight) *plus* `server.closeIdleConnections()` — keep-alive sockets otherwise stall the close — and after a grace period `server.closeAllConnections()`; (3) deadline the whole thing: `setTimeout(() => process.exit(1), 10_000).unref()`; (4) close pools (DB, Redis) only after traffic stops; (5) exit 0. Any `setInterval` or handle without `.unref()` blocks natural exit — the cause of most "my process won't die" reports. Test it: SIGTERM under load in CI, assert zero dropped responses.
- **Docker PID 1 problem.** `CMD npm start` makes npm PID 1, and npm does not reliably forward SIGTERM — your shutdown code never runs and Docker SIGKILLs at the timeout. Exec-form `CMD ["node", "server.js"]`, or `--init`/tini.
- **`Promise.all` on side-effecting fan-out.** First rejection returns control while sibling writes continue unsupervised — partial-state plus possible late unhandled rejections. Use `Promise.allSettled` and inspect results, or wire an `AbortSignal` through so siblings actually stop.
- **Timers pile-up.** `setInterval` whose callback can outlast the interval overlaps itself (async callback + slow downstream). Use recursive `setTimeout` scheduled at completion, or a loop with `await setTimeout(ms)` from `node:timers/promises`.
- **npm hygiene (post-Shai-Hulud, non-negotiable):**
  - Commit lockfiles; install with `npm ci` (never bare `npm install` in CI).
  - `ignore-scripts=true` in `.npmrc` — lifecycle scripts are the #1 initial-execution vector; allowlist the few genuinely needed builds (e.g. `@lavamoat/allow-scripts`).
  - Enforce a **cooldown window**: don't auto-merge dependency bumps published <7 days ago — the 2025–26 registry compromises were typically detected within days (pnpm `minimumReleaseAge`, Renovate `minimumReleaseAge` implement this).
  - Pin exact versions for applications' direct deps; review lockfile diffs in PRs (a lockfile-only change touching registries/URLs is a red flag).
  - `npm audit` alone is noise; behavior/signature scanners (socket.dev-class) catch malware CVE feeds miss. Never `npx <unvetted>` in CI with credentials in env — use `--ignore-scripts` and pinned versions there too.
- **`NODE_ENV=production` still matters** — express view caching and many libraries' fast paths key on it; forgetting it is a real 2x on express apps. Fastify doesn't care, but your dependencies might.
- **Reading `heapUsed` as "memory used by my app."** V8 heap ≠ RSS: Buffers live outside the heap (`external`), and RSS includes fragmentation. Alerting on the wrong number produces phantom leaks and missed real ones.
- **Container memory limits vs V8 defaults.** V8's old-space default doesn't know about your cgroup limit; a 512MB container with default heap sizing gets OOM-killed by the kernel before V8 ever feels pressure (no GC effort, no heap snapshot on the way down). Set `--max-old-space-size` to ~75-80% of the container limit so V8 GCs hard (or fails with a JS heap error you can capture) instead of being SIGKILLed.
- **`await` in a loop when you meant parallel — and vice versa.** Sequential `for (const u of users) await notify(u)` where order doesn't matter wastes wall time; but the "fix" `Promise.all(users.map(notify))` with 50k users self-DoSes the downstream. The production shape is bounded concurrency: `p-limit`, a worker-pool pattern, or batching. Unbounded `Promise.all` fan-out is a bug, not an optimization.
- **Body parsing without limits.** Default-configured JSON body parsing with no size cap is a one-request memory bomb and a sync-parse stall. Set body size limits at the framework (fastify `bodyLimit`) or proxy layer; reject early.
- **Keep-alive mismatch 502s.** If the server's `keepAliveTimeout` is shorter than the load balancer's idle timeout, the server closes a socket the LB just reused → intermittent 502/ECONNRESET. Set server `keepAliveTimeout` (and `headersTimeout` above it) longer than the LB idle timeout.

## Worked micro-examples

**Backpressure-correct streaming endpoint (fastify):**
```ts
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

app.get('/export', async (req, reply) => {
  const rows = db.streamRows('SELECT ...');          // Readable (object mode)
  reply.raw.writeHead(200, {
    'content-type': 'application/x-ndjson',
    'content-encoding': 'gzip',
  });
  await pipeline(
    rows,
    async function* (src) {                          // async-generator transform
      for await (const r of src) yield JSON.stringify(r) + '\n';
    },
    createGzip(),
    reply.raw,                                       // slow client ⇒ gzip pauses ⇒ DB cursor pauses
  );
  return reply;                                      // tells fastify the reply was handled raw
});
```

**Event-loop lag guardrail (cheap, always-on):**
```ts
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  const p99ms = h.percentile(99) / 1e6;              // histogram is in nanoseconds
  metrics.gauge('event_loop_p99_ms', p99ms);
  if (p99ms > 100) log.warn({ p99ms }, 'event loop blocked; take a CPU profile');
  h.reset();
}, 10_000).unref();                                  // unref: never block shutdown
```

**Graceful shutdown skeleton:**
```ts
let shuttingDown = false;
app.get('/healthz/ready', (_req, reply) => reply.code(shuttingDown ? 503 : 200).send());

process.on('SIGTERM', async () => {
  shuttingDown = true;                               // 1. fail readiness; LB starts draining
  setTimeout(() => process.exit(1), 10_000).unref(); // 3. hard deadline
  await app.close();                                 // 2. fastify: stops listening, waits in-flight,
                                                     //    runs onClose hooks (close DB pools here — 4)
  process.exit(0);                                   // 5
});
process.on('unhandledRejection', (err) => { log.fatal(err); process.exit(1); }); // crash-only
```

**CPU work off the loop with a worker pool (piscina):**
```ts
// worker.ts — pure function, no app state
import { createHash } from 'node:crypto';
export default function thumbnail({ buf }: { buf: ArrayBuffer }): ArrayBuffer {
  // ... sharp/resize etc. Return transferable, don't structured-clone megabytes.
  return buf;
}

// server.ts
import Piscina from 'piscina';
const pool = new Piscina({
  filename: new URL('./worker.js', import.meta.url).href,
  maxThreads: 4,                       // size to cores minus loop headroom; measure, don't max
});
app.post('/thumbnail', async (req, reply) => {
  const buf = await req.file();        // stream to buffer with a size cap
  const out = await pool.run({ buf: buf.buffer }, { transferList: [buf.buffer] }); // transfer, not copy
  reply.type('image/webp').send(Buffer.from(out));
});
```

**Supply-chain guardrails (.npmrc + CI):**
```ini
# .npmrc — checked into the repo
ignore-scripts=true          # no lifecycle-script execution on install (allowlist exceptions)
save-exact=true              # applications pin direct deps exactly
```
```yaml
# CI install step
- run: npm ci --ignore-scripts        # lockfile-exact, scripts still disabled
- run: npx --yes @lavamoat/allow-scripts@<pinned> run   # run ONLY allowlisted build scripts
```
Plus: Renovate/dependabot configured with a ≥7-day `minimumReleaseAge` cooldown, and lockfile diffs reviewed like code — a dependency bump that edits resolved URLs or adds install scripts is a stop-the-line signal.

## Verification and stopping rule

Before presenting Node backend advice or code:
1. Walk every stream path and *name* its backpressure mechanism (pipeline? drain handling?). Can't name it → it's a memory bomb under a slow consumer.
2. Walk every promise: someone awaits it or explicitly handles it, including on error branches and deferred-await patterns.
3. Confirm shutdown: SIGTERM handled, node is PID 1, readiness flips, idle connections closed, timers `.unref()`ed, deadline set.
4. For any performance claim, require a flamegraph or loop-delay number — "fastify is faster" is true but irrelevant if profiles show your time in Postgres.
5. Version-gate: permission model stable on 24; `require(esm)` and `import.meta.dirname` on 22+; don't cite Node-26-only APIs for LTS deployments.
6. For dependency advice, check the package's publish recency and maintenance before recommending it — and never suggest running install scripts from an unvetted package.

Stopping rule: done when loop delay is instrumented, rejections crash loudly, shutdown is tested under load, and installs run scripts-disabled + lockfile + cooldown. Beyond that, framework rewrites for benchmark deltas (express→fastify→hono for its own sake) are waste unless HTTP overhead actually appears in your profiles.
