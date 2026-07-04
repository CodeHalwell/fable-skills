---
name: concurrency-and-parallelism
description: Load when writing or reviewing concurrent/parallel/async code — threads, locks, atomics, async/await, channels/actors, thread pools — or diagnosing races, deadlocks, and event-loop stalls. Also load for questions about Python's GIL, memory models, pool sizing, or choosing between shared memory and message passing.
---

# Concurrency and Parallelism

## Core mental model

- **Data race ≠ race condition; you must fix both, and fixing one doesn't fix the other.**
  - A *data race* is two threads accessing the same memory unsynchronized with at least one write — undefined behavior in C/C++ (and Go, and unsafe Rust); torn or stale values elsewhere.
  - A *race condition* is a correctness bug from operation *ordering*, and it survives full synchronization: `if key not in cache: cache[key] = compute()` over a lock-per-operation dict is data-race-free and still a check-then-act race.
  - Locks and atomics remove data races. Only redesigning the *protocol* — compound atomic operations, CAS loops, single-writer ownership — removes race conditions.
- **Reason in happens-before, not in time.** Without a synchronization edge (mutex release→acquire, channel send→receive, thread start/join, acquire/release atomics), another thread may observe your writes reordered — or never. Compilers and CPUs both reorder aggressively. "Thread B runs later, so it sees A's write" is false without an edge. Every shared datum needs an answer to one question: *which edge publishes it to its readers?*
- **Prefer moving data ownership over sharing it.** The cheapest concurrency bug is the one made impossible: single-writer designs, message passing that *transfers* ownership, immutable snapshots, and thread confinement eliminate bug classes instead of guarding them. Locks are for when ownership genuinely must be shared — a tool of last resort with a discipline attached, not the default.
- **Concurrency ≠ parallelism.** Concurrency is structure (dealing with many things at once); parallelism is execution (doing many things at once). Async/await provides concurrency on one core — it helps I/O-bound waiting and does *nothing* for CPU-bound work. Threads/processes provide parallelism. Choosing async to speed up computation, or thread-per-request to fix I/O-wait cost at 10k connections, is a category error.
- **Every unbounded queue is a lie about capacity.** A producer faster than its consumer means either a bounded queue that pushes back or memory growth until OOM at peak load — exactly when you can least afford it. Backpressure is a design requirement: decide explicitly what happens when full (block, shed, coalesce), don't let the allocator decide for you.

## Decision frameworks

### Concurrency model selection
| Workload | Model | Why |
|---|---|---|
| Many concurrent I/O waits in one runtime (servers, scrapers, API fan-out) | async/await or lightweight tasks (asyncio, Node, Go goroutines) | 10k+ concurrent waits at trivial memory; OS threads cap around thousands (MB-scale stacks, scheduler overhead) |
| CPU-bound parallelism in Python | `concurrent.futures.ProcessPoolExecutor` / `multiprocessing`, or push work into GIL-releasing native code (NumPy, polars) | Threads give ~zero CPU parallelism under the GIL |
| CPU-bound in Go/Rust/Java/C# | Thread pool sized ≈ core count | Real parallelism; extra threads beyond cores only add context-switch cost |
| Pipeline stages, fan-out/fan-in, cancellation trees | Channels/CSP or structured concurrency (Go channels, `asyncio.TaskGroup`, Trio nurseries) | Ownership transfer plus explicit topology; the deadlock surface is the channel graph, which you can draw and audit |
| Stateful entities with independent lifecycles (sessions, devices, game objects) | Actors — one mailbox, one owner per entity | Serializes access per entity without locks; scales with entity count |
| Read-mostly shared config/lookup tables | Immutable snapshot swapped atomically (`AtomicReference`, RCU-style) | Readers take zero locks; writer copies, mutates, publishes. Beats a RWLock on both speed and correctness |
| Genuinely shared hot mutable state (counters, small maps) | Locks — or per-thread sharding aggregated on read (`LongAdder` pattern) | Sharding beats one contended atomic; *contention*, not locking, is the real cost |

Channels vs shared memory rule: channels when the design is *data flowing through stages with clear ownership transfer*; shared memory + locks when many parties need random access to one structure. Forcing random-access patterns through channels produces a slow, deadlock-prone reimplementation of a lock.

### Lock discipline (when you do lock)
- **Granularity = one lock per named invariant.** Not lock-per-field (invariants spanning two fields get torn between acquisitions) and not one-lock-for-everything (contention). Write the invariant each lock guards in a comment next to its declaration; if you can't state it, the lock is decorative.
- **Deadlock prevention is a global acquisition order:** establish a total order over locks (by layer, by address, by entity ID) and only ever acquire in that order. Every code path taking 2+ locks must be checkable against the order.
- **Never call unknown code while holding a lock** — callbacks, listeners, logging that might flush, `__eq__`/`hashCode` on user-supplied types, signal emission. That code may take another lock (ordering violation → deadlock) or reenter yours. Snapshot state under the lock; invoke the callback outside it.
- **Hold time:** compute nothing, await nothing, do no I/O under a lock. Copy in, compute outside, write back under re-acquired lock with revalidation (or CAS).
- **Bounded-queue interaction:** holding lock L while doing a blocking `put` on a bounded queue whose consumer needs L is a deadlock with two moving parts nobody sees in review. Any blocking operation under a lock deserves a comment justifying it — the default answer is no.

### Thread-pool sizing math
- CPU-bound: threads = cores (±1). More only adds switching overhead.
- Mixed blocking: threads ≈ cores × (1 + wait_time/compute_time). Example: 8 cores, tasks spend 90ms waiting on I/O per 10ms of CPU → 8 × (1 + 9) = 80 threads to saturate the cores. If the formula outputs an absurd number (thousands), that's the signal to switch to async rather than spawn.
- **Bulkheads:** separate bounded pools per external dependency. One pool shared by "call slow service X" and "serve all requests" lets X's brownout consume every thread — thread-pool exhaustion is the classic cascading-failure mechanism. Dedicated pool + bounded queue + timeout per dependency.
- **Little's law sanity check:** required concurrency = arrival rate × latency. 500 req/s × 0.2s = 100 in-flight; a 50-thread synchronous server *cannot* serve it regardless of CPU headroom. Run this arithmetic before tuning anything else.

### async/await pitfalls (Python-flavored; same shapes in JS/C#)
- **Blocking the loop:** one synchronous call — `requests.get`, `time.sleep`, a heavy CPU loop, a `psycopg2` query — freezes *every* task on the loop. Rules: inside `async def`, only awaitable I/O, trivial CPU, or explicit offload via `await asyncio.to_thread(fn)` / `run_in_executor`. Detection: `asyncio.run(main(), debug=True)` logs callbacks over 100ms; run a loop-lag watchdog metric in production.
- **Forgotten await / lost tasks:**
  - Calling `coro()` without `await` does nothing; Python's "coroutine was never awaited" warning should be treated as an error in CI (`-W error::RuntimeWarning` in tests catches it).
  - `asyncio.create_task(...)` without keeping a reference can be garbage-collected mid-flight. Keep handles (a set with a done-callback that discards) or use `asyncio.TaskGroup`, which also propagates exceptions.
  - A bare task whose exception is never retrieved fails *silently* until interpreter shutdown prints a cryptic message. Every spawned task needs an owner.
- **Cancellation:** every `await` is a cancellation point; `CancelledError` can surface at any of them.
  - Cleanup goes in `finally` or an `async with` context manager; cleanup that itself awaits can be re-cancelled — shield it (`asyncio.shield`) or use the pattern your framework provides.
  - Never swallow `CancelledError`: it's a `BaseException`, so `except Exception` correctly misses it, but bare `except:` and `except BaseException:` swallow it and silently break every timeout and task group upstream. Re-raise it if you must intercept.
- **Cooperative means cooperative:** a tight async loop that never awaits starves all peers. `await asyncio.sleep(0)` yields explicitly when chunking semi-CPU work you can't offload.
- **Sync and async colored functions don't mix silently:** calling an async API from sync code needs `asyncio.run`/a running loop; blocking on `loop.run_until_complete` from *inside* a coroutine deadlocks. Design libraries to pick one color per layer.

### Python GIL realities
- One interpreter lock, one bytecode-executing thread at a time: threads give **zero** speedup for pure-Python CPU work, often a mild slowdown from handoff churn.
- Threads *do* parallelize whatever releases the GIL: all blocking I/O, plus C extensions that release it around long operations — NumPy kernels, `hashlib` on large buffers, zlib, most database drivers, `bcrypt`.
- Therefore: I/O-bound → threads are fine (or async at high fan-out); CPU-bound → processes (`ProcessPoolExecutor`; mind pickling cost of arguments and results — chunk work so IPC amortizes) or native/vectorized code.
- **The GIL does not make your code thread-safe.** It makes individual bytecodes atomic-ish; `x += 1`, `d[k] = d.get(k, 0) + 1`, and every check-then-act sequence interleave and corrupt. You still need locks/queues for compound operations. Free-threaded (no-GIL) builds remove even the bytecode-level accident — never rely on it.

### Lock-free: when and when not
- Legitimate uses: hot counters and flags (`fetch_add`), publish-once pointer swaps (immutable snapshot pattern), and established library structures (`java.util.concurrent`, `crossbeam`, `folly`). Also when priority inversion or signal/interrupt context forbids locks.
- **Never hand-roll linked lock-free structures.** ABA, safe memory reclamation (hazard pointers/epochs), and memory-ordering subtleties defeat almost everyone, and the failure reproduces only under production contention. Use a library or use a lock.
- "Lock-free" means guaranteed system-wide progress, not "faster": under real contention a CAS retry loop can burn more CPU than a well-held mutex. Benchmark under realistic contention before choosing it — an uncontended mutex costs ~tens of nanoseconds and is boring, which is a feature.

## Failure modes & pitfalls

- **Check-then-act on shared state.** `if not path.exists(): create()`, `if balance >= amt: balance -= amt`, singleton `if instance is None: instance = ...` — state changes between the check and the act. Fix with one atomic compound operation: `dict.setdefault`, `INSERT ... ON CONFLICT`, compare-and-swap, `open(..., "x")`/`O_EXCL`, or one lock held across check *and* act.
- **Guarding writes but not reads.** Locking mutations "because reads are safe" — readers see torn or mid-rebalance state (a dict resizing, a tree rotating), and without an acquire edge possibly stale values forever. The lock or snapshot-publication must cover readers too; "it's just a read" is not a memory-model argument.
- **Atomic/volatile as a lock substitute.** Atomicity of a single load or store doesn't make read-modify-write atomic: two threads doing atomic-read, add, atomic-write still lose updates. You need `fetch_add`/CAS or a lock. Same trap as the GIL one — single-operation atomicity never implies compound-operation atomicity.
- **Deadlock via the invisible second lock.** The production deadlock is rarely two explicit mutexes; it's your lock plus the logging handler's lock, a GC finalizer, a `synchronized toString()`, or a blocking put on a bounded queue whose consumer needs your lock. Audit everything callable under each lock. Diagnosis: a thread dump (`py-spy dump`, `jstack`, `kill -QUIT`) names both holders instantly — *always take dumps before restarting a hung process*; the evidence dies with the restart.
- **The async function that's secretly synchronous.** An `async def` doing CPU work or calling a sync driver stalls the entire service — and passes all tests, because single-request tests never observe loop starvation. Review rule: every operation inside `async def` is awaited non-blocking I/O, trivial CPU, or explicit offload. Verify under concurrent load with a loop-lag metric.
- **Fire-and-forget without exception routing.** Background work (`create_task`, `Thread(daemon=True)`, executor `submit` whose Future nobody reads) dies silently. `ThreadPoolExecutor.submit` stores the exception until `.result()` is called — iterate the futures and call it. Every spawned unit needs a place its exception lands: TaskGroup/nursery, a done-callback that logs, or a join that inspects results.
- **Sharing non-thread-safe clients across threads/tasks.** One `sqlite3` connection across threads, an SQLAlchemy `Session` shared by workers, one DB transaction object used concurrently — corruption or crosstalk that appears only under load. Rule: one client/session per thread or task, or an explicitly thread-safe pool. Check the library's documented thread-safety, don't assume.
- **Timeout-free blocking calls.** Any lock acquire, `queue.get`, `join`, or network read without a timeout is a hang waiting for its trigger. Production rule: every blocking call carries a timeout and a defined behavior on expiry (retry, shed, escalate).
- **Testing concurrency with sleeps.** `time.sleep(0.1)` "to let the other thread run" produces tests that pass on your laptop and flake in CI forever. Instead: force the interleaving with `threading.Event`/barriers/latches; stress-loop the race body thousands of times; run race detectors — `go test -race` (non-negotiable in Go), TSan for C/C++, `loom` for Rust lock-free logic, asyncio debug mode for Python.
- **Cleanup after the await instead of in `finally`.** On timeout the caller cancels you mid-await; a connection/file/lock released on the line *after* the await leaks. Acquire with `async with`/`with` wherever the resource supports it; otherwise `try/finally` from the moment of acquisition.
- **Assuming FIFO or exactly-once anywhere it isn't promised.** Thread wakeups aren't FIFO, queue consumers interleave, and most delivery systems are at-least-once. Handlers must be idempotent and order-tolerant unless the specific mechanism documents otherwise.

## Worked micro-examples

### 1. The GIL-safe-looking counter that isn't
```python
import threading
counter = 0
def bump():
    global counter
    for _ in range(1_000_000):
        counter += 1          # LOAD, ADD, STORE — three interleavable steps
threads = [threading.Thread(target=bump) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)                # far below 4_000_000, different every run
```
Lost updates despite the GIL: the interpreter can hand off between the load and the store. Fixes, best first: (1) restructure so each thread owns a local count, summed after join — no sharing, no bug, no lock; (2) one `threading.Lock` around the increment; (3) a per-operation lock on a shared counter is the worst-performing correct option. General lesson: *prove* compound-op atomicity; never infer it from "GIL" or "atomic type."

### 2. Deadlock by lock ordering, and the fix
```python
def transfer(src, dst, amt):
    with src.lock:                     # T1: transfer(a, b) holds a.lock
        with dst.lock:                 # T2: transfer(b, a) holds b.lock
            src.bal -= amt             # both wait forever
            dst.bal += amt

def transfer(src, dst, amt):           # fix: global acquisition order
    first, second = (src, dst) if src.id < dst.id else (dst, src)
    with first.lock, second.lock:
        src.bal -= amt
        dst.bal += amt
```
Verification habit: for any code taking two or more locks, name the global order, then check every acquisition site against it. A site where the order can't be known at acquisition time must be redesigned — single lock, single owner, or trylock-with-backoff.

### 3. Event-loop stall, quantified
An asyncio service handles 200 concurrent requests at p99 = 30ms. Someone adds `bcrypt.hashpw` (~200ms of CPU) inline in an `async def` login handler. At just 5 logins/s × 200ms, the loop is 100% occupied: *every* request — health checks included — queues behind hashing, p99 goes to seconds service-wide, and the failing health check gets the instance killed, shifting load to its neighbors (cascade).
Fix: `await asyncio.to_thread(bcrypt.hashpw, pw, salt)` — bcrypt releases the GIL, so a thread genuinely parallelizes it; use a process pool if the work were pure-Python CPU. Diagnostic tell for this whole class: latency degrades *globally* (unrelated endpoints too) in multiples of one operation's duration.

## Self-check before presenting concurrent code

- For every shared mutable datum: name its owner, or the lock/edge that guards it — *including all readers*. Anything unnamed is a bug you haven't met yet.
- For every check-then-act or read-modify-write on shared state: point to the mechanism making the compound operation atomic (lock span, CAS, `setdefault`, DB constraint).
- Locks: is there a stated global acquisition order? Is anything awaited, blocking, I/O-bound, or user-callable executed while held?
- Async: zero blocking calls inside any `async def`? Every spawned task owned (TaskGroup/nursery/handle + exception sink)? Every cleanup path cancellation-safe (`finally`/`async with`)? `CancelledError` never swallowed?
- Every queue bounded, with a stated full-behavior? Every blocking call timeboxed?
- Pool sizes justified by the arithmetic (cores × (1 + wait/compute); Little's law), with bulkheads per external dependency?
- Did it run under a race detector or debug mode (`-race`, TSan, loom, asyncio debug) and a stress loop with forced interleavings — not just once, green, on a warm laptop?
