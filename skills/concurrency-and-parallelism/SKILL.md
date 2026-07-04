---
name: concurrency-and-parallelism
description: Load when writing or reviewing concurrent/parallel/async code — threads, locks, atomics, async/await, channels/actors, thread pools — or diagnosing races, deadlocks, and event-loop stalls. Also load for questions about Python's GIL, memory models, pool sizing, or choosing between shared memory and message passing.
---

# Concurrency and Parallelism

## Core mental model

- **Data race ≠ race condition; you must fix both, and fixing one doesn't fix the other.** A *data race* is two threads touching the same memory unsynchronized with at least one write — undefined behavior in C/C++/Go/Rust-unsafe, torn or stale values elsewhere. A *race condition* is a correctness bug from operation *ordering*, and it survives full synchronization: `if key not in cache: cache[key] = compute()` under a lock-per-operation dict is data-race-free and still a check-then-act race. Atomics and locks remove data races; only redesigning the *protocol* (compound atomic operations, CAS loops, single-writer ownership) removes race conditions.
- **Reason in happens-before, not in time.** Without a synchronization edge (mutex release→acquire, channel send→receive, thread start/join, acquire/release atomics), another thread may observe your writes in a different order or never — compilers and CPUs both reorder. "Thread B runs later, so it sees A's write" is false without an edge. Every shared datum needs an answer to: *which edge publishes it?*
- **Prefer moving data ownership over sharing it.** The zero-cost concurrency bug is the one made impossible: single-writer designs, message passing that *transfers* ownership, immutable snapshots (copy-on-write), and thread confinement eliminate whole bug classes rather than guarding them. Locks are for when ownership genuinely must be shared — a last resort with a discipline attached, not the default tool.
- **Concurrency (structure: dealing with many things at once) ≠ parallelism (execution: doing many things at once).** Async/await gives concurrency on one core — it helps I/O-bound waiting, and does nothing for CPU-bound work. Threads/processes give parallelism. Choosing async to speed up computation, or thread-per-request to fix I/O wait cost, is a category error.
- **Every unbounded queue is a lie about capacity.** Backpressure is not optional: producers faster than consumers means either bounded queues that push back (blocking/erroring at submit) or memory growth until OOM at peak load — precisely when you can least afford it. Design the "what happens when full" answer explicitly (block, shed, coalesce).

## Decision frameworks

### Concurrency model selection
| Workload | Model | Why |
|---|---|---|
| Many concurrent I/O waits, single language runtime (servers, scrapers) | async/await or lightweight tasks (asyncio, Node, Go goroutines) | 10k+ concurrent waits at trivial memory; threads cap at ~thousands (MB-scale stacks, scheduler cost) |
| CPU-bound parallel work in Python | `multiprocessing` / `concurrent.futures.ProcessPoolExecutor`, or push into GIL-releasing native code (NumPy, polars) | Threads give ~zero CPU parallelism under the GIL (see below) |
| CPU-bound in Go/Rust/Java/C# | Thread pool sized ≈ cores | Real parallelism; more threads than cores just adds context-switch cost |
| Pipeline stages, fan-out/fan-in, cancellation trees | Channels/CSP or structured tasks (Go channels, `asyncio.TaskGroup`, Trio nurseries) | Ownership transfer + explicit topology; deadlock surface is the channel graph, which you can draw |
| Stateful entities with independent lifecycles (sessions, devices, game objects) | Actors (one mailbox, one owner per entity) | Serializes access per entity without locks; scales by entity count |
| Shared read-mostly config/lookup tables | Immutable snapshot swapped atomically (`AtomicReference`, RCU-style) | Readers take zero locks; writer copies + publishes. Beats RWLock in both speed and correctness |
| Genuinely shared hot mutable state (counters, small maps) | Locks — or per-thread sharding with aggregation (`LongAdder` pattern) | Sharding beats one contended atomic; contention, not locking, is the cost |

Channels vs shared memory rule: channels when you can express the design as *data flowing through stages with clear ownership transfer*; shared memory + locks when many parties need random-access to one structure. Forcing random-access patterns through channels yields a slow, deadlock-prone lock reimplementation.

### Lock discipline (when you do lock)
- **Granularity:** one lock protecting a named *invariant*, not "a lock per field" (invariants spanning two fields get torn) and not "one lock for everything" (contention). Document what invariant each lock guards, next to its declaration.
- **Deadlock prevention is a global ordering:** establish a total order on locks (by layer, by address, by ID) and only acquire in that order. `transfer(a, b)` must sort: `first, second = (a, b) if a.id < b.id else (b, a)` then lock `first`, `second` — otherwise two opposite transfers deadlock. Alternatives when ordering is impossible: `trylock` with backoff, or restructure to a single lock/owner.
- **Never call unknown code while holding a lock** — callbacks, listeners, logging that might flush, `__eq__` on user types. That code may take another lock (ordering violation → deadlock) or reenter yours. Snapshot under the lock, call outside it.
- Hold time: compute nothing, await nothing, I/O nothing under a lock. Copy in, compute out, copy back with revalidation (or CAS).

### Thread-pool sizing math
- CPU-bound: N_threads = N_cores (± 1). More adds overhead only.
- Mixed blocking: N_threads ≈ N_cores × (1 + wait_time/compute_time). Example: 8 cores, tasks wait 90ms on I/O per 10ms of CPU → 8 × (1 + 9) = 80 threads to keep cores busy. If that number comes out absurd (thousands), that's the signal to go async instead of spawning.
- Separate pools per dependency class: one pool shared by "call slow service X" and "serve requests" lets X's brownout starve everything (thread-pool exhaustion is the classic cascading-failure mechanism). Bulkhead: dedicated bounded pool per external dependency + bounded queue + timeout.
- Little's law for sanity: concurrency needed = arrival_rate × latency. 500 req/s × 0.2s = 100 in-flight; a 50-thread synchronous server *cannot* meet that regardless of CPU.

### async/await pitfalls (Python-flavored; same shapes in JS/C#)
- **Blocking the loop:** one synchronous call — `requests.get`, `time.sleep`, heavy CPU, `psycopg2` query — freezes *every* task on the loop. Rules: only awaitable I/O in async code; CPU/blocking work goes through `await loop.run_in_executor(None, fn)` / `asyncio.to_thread(fn)`. Detect: `asyncio.run(main(), debug=True)` logs callbacks >100ms; watchdog-style loop-lag metrics in prod.
- **Forgotten await / lost tasks:** calling `coro()` without `await` does nothing (Python warns "coroutine was never awaited" — treat as error). `asyncio.create_task(...)` without keeping a reference can be *garbage-collected mid-flight* — keep task handles (a set with done-callback discard) or use `asyncio.TaskGroup`, which also propagates exceptions. A bare `create_task` whose exception is never retrieved fails **silently** until process exit.
- **Cancellation:** every `await` is a cancellation point; `CancelledError` can surface at any of them. Cleanup must be in `finally`, and `finally` blocks that themselves `await` must be shielded or they get re-cancelled (`asyncio.shield`, or in modern Python check `asyncio.current_task().cancelling()`). Never swallow `CancelledError` broadly — `except Exception` doesn't catch it (it's `BaseException`) but `except BaseException` and bare `except` do, and swallowing it breaks timeouts and task groups upstream.
- **Async is cooperative:** a tight `async` loop that never awaits starves peers. `await asyncio.sleep(0)` yields explicitly when chunking CPU-ish work you can't offload.

### Python GIL realities
- One interpreter, one bytecode-executing thread at a time: threads give **zero** speedup for pure-Python CPU work, and often mild slowdown (GIL handoff churn). Threads *do* help when the work releases the GIL: all blocking I/O, plus C extensions that release it (NumPy ops, hashlib on large buffers, zlib, most DB drivers).
- Consequently: I/O-bound → threads are fine (or async at high fan-out); CPU-bound → processes (`ProcessPoolExecutor` — mind pickling cost of arguments/results; chunk work so IPC amortizes) or native/vectorized code.
- The GIL does **not** make your code thread-safe. It makes single *bytecodes* atomic-ish; `x += 1`, `d[k] = d.get(k, 0) + 1`, check-then-act sequences all interleave and corrupt. You still need locks/queues for compound operations. (And free-threaded/no-GIL builds remove even the bytecode-level accident — never rely on it.)

### Lock-free: when and when not
- Reach for lock-free (CAS loops, atomic counters, established concurrent structures) only for: hot counters/flags, publish-once pointers (immutable snapshot swap), or when priority-inversion/deadlock constraints forbid locks. Use library structures (`java.util.concurrent`, `crossbeam`, `folly`) — never hand-roll linked lock-free structures; ABA, memory reclamation (hazard pointers/epochs), and memory-ordering subtleties defeat almost everyone, and the bug reproduces only under production load.
- "Lock-free" means system-wide progress, not "faster": under contention a CAS retry loop can burn more CPU than a mutex. Benchmark under realistic contention before choosing it.

## Failure modes & pitfalls

- **Check-then-act on shared state.** `if not path.exists(): create()`, `if balance >= amt: balance -= amt`, singleton `if instance is None:` — the state can change between check and act. Fix with a single atomic compound op: `dict.setdefault`, `INSERT ... ON CONFLICT`, CAS, `O_EXCL` file create, or hold one lock across check *and* act.
- **Guarding writes but not reads.** Locking mutations of a structure while reading it lock-free "because reads are safe" — reads see torn/mid-rebalance state (a `dict` resizing, a list growing), and without an acquire edge, possibly *stale* values forever. The lock (or snapshot-publication) must cover readers too.
- **`volatile`/atomic as a lock substitute.** Atomicity of a single load/store doesn't make read-modify-write atomic: two threads doing atomic-read, +1, atomic-write still lose updates. Need `fetch_add`/CAS or a lock. Same trap as the GIL one — single-op atomicity ≠ compound-op atomicity.
- **Deadlock via invisible second lock.** The classic isn't two explicit mutexes — it's lock + logging handler lock, lock + GC finalizer, lock + `synchronized` toString, lock held across `queue.put` on a *bounded* queue whose consumer needs that lock. Audit everything callable under each lock. Diagnosis: thread dump (`py-spy dump`, `jstack`) shows both holders instantly — take dumps before restarting a hung process.
- **Async function that's secretly synchronous.** `async def` that does CPU work or calls a sync driver stalls the whole service, and it passes all tests (single-request tests never see loop starvation). Review rule: inside `async def`, every operation is either `await`ed non-blocking I/O, trivial CPU, or explicitly offloaded. Load-test with concurrent requests, watch loop lag.
- **Fire-and-forget without exception routing.** Background tasks (`create_task`, `Thread(daemon=True)`, executor `submit` whose Future nobody reads) that die silently. Every spawned unit needs a place its exception lands: TaskGroup/nursery, done-callbacks that log, or joining with result inspection. `ThreadPoolExecutor.submit` swallows exceptions until `.result()` — `map` over futures without collecting results hides crashes.
- **Sharing non-thread-safe clients across threads/tasks.** One `sqlite3` connection, one `requests.Session` (mostly ok) vs one DB transaction object (never ok), botocore clients pre-check, ORM sessions (`SQLAlchemy Session` is not thread-safe) shared across workers — corruption or crosstalk under load only. Rule: one client per thread/task, or an explicitly thread-safe pool.
- **Timeout-free blocking calls.** Any lock acquire, queue get, join, or network read without a timeout is a hang waiting for its trigger. Production rule: every blocking call has a timeout + a defined behavior on expiry.
- **Testing concurrency with sleeps.** `sleep(0.1)` "to let the other thread run" makes flaky tests that pass on the laptop and fail in CI. Use events/barriers to force the interleaving you're testing (`threading.Event`, latches), run the race body thousands of times in a stress loop, and use race detectors: `go test -race` (non-negotiable in Go), TSan for C/C++/Rust, loom for Rust lock-free logic.
- **Ignoring cancellation in cleanup paths, then leaking.** On timeout, callers cancel your task mid-`await`; if the connection/file/lock release lives after the await rather than in `finally` (or an async context manager), it leaks. Always acquire-with-`async with` where available.

## Worked micro-examples

**1. The GIL-safe-looking counter that isn't.**
```python
import threading
counter = 0
def bump():
    global counter
    for _ in range(1_000_000):
        counter += 1          # read, +1, write — three interleavable steps
threads = [threading.Thread(target=bump) for _ in range(4)]
[t.start() for t in threads]; [t.join() for t in threads]
print(counter)                # ≪ 4_000_000, varies per run
```
Lost updates despite the GIL: `+=` compiles to LOAD/ADD/STORE, and the GIL can hand off between them. Fixes, best-first: restructure so each thread owns a local count and you sum at join (no sharing → no bug); else one `threading.Lock` around the increment; a per-op lock on a shared int is the worst-performing correct option. The general lesson: prove compound-op atomicity, never infer it from "GIL" or "atomic type."

**2. Deadlock by lock ordering, and the fix.**
```python
def transfer(src, dst, amt):
    with src.lock:              # T1: transfer(a,b) holds a.lock
        with dst.lock:          # T2: transfer(b,a) holds b.lock → both wait forever
            src.bal -= amt; dst.bal += amt

def transfer(src, dst, amt):    # fix: global acquisition order by stable ID
    first, second = (src, dst) if src.id < dst.id else (dst, src)
    with first.lock, second.lock:
        src.bal -= amt; dst.bal += amt
```
Verification habit: for any code taking 2+ locks, name the global order and check every acquisition site follows it; any site that can't (order unknowable at that point) must be redesigned to single-lock or trylock-retry.

**3. Event-loop stall, quantified.** An asyncio service handles 200 concurrent requests fine at p99 = 30ms. Someone adds `bcrypt.hashpw` (≈200ms CPU) inline in an `async def` login handler. Now *every* concurrent request — health checks included — queues behind each login: 5 logins/s × 200ms = the loop is 100% blocked; p99 goes to seconds service-wide, and the health check timing out gets the instance killed. Fix: `await asyncio.to_thread(bcrypt.hashpw, pw, salt)` (bcrypt releases the GIL, so threads genuinely parallelize it) — or a process pool if the work were pure-Python CPU. The diagnostic tell: latency degrades *globally* and in multiples of one operation's duration.

## Self-check before presenting concurrent code

- For every shared mutable datum: name its owner, or the lock/edge that guards it, *including all readers*. Anything unnamed is a bug.
- For every check-then-act or read-modify-write on shared state: is the compound operation atomic (one lock span, CAS, DB constraint)? Point to the mechanism.
- Locks: is there a stated global order? Is anything awaited, blocking, or user-callable while held?
- Async: does every `async def` contain zero blocking calls? Is every spawned task owned by a TaskGroup/nursery or has an exception sink? Does every `finally` survive cancellation?
- Every queue bounded with a stated full-behavior? Every blocking call timeboxed?
- Did it run under a race detector / debug mode (`-race`, TSan, `asyncio` debug) and a stress loop with forced interleavings — not just once, green, on a warm laptop?
