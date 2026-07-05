---
name: concurrency-and-parallelism
description: Load when writing or reviewing concurrent/parallel/async code — threads, locks, atomics, async/await, channels/actors, thread pools — or diagnosing races, deadlocks, and event-loop stalls. Also load for questions about Python's GIL, memory models, pool sizing, or choosing between shared memory and message passing.
---

# Concurrency and Parallelism — correction sheet

Baseline model knowledge already covers the textbook layer well (data race vs. race condition, GIL mechanics, Goetz pool sizing, double-checked locking, fork+threads, lock-free caveats, Little's law). This sheet keeps only the review rules and operational specifics that don't surface unprompted.

## Rules that don't surface unprompted

- **Blocking put under a lock is the invisible deadlock.** Holding lock L while doing a blocking `put` on a bounded queue whose *consumer* needs L deadlocks with two moving parts nobody sees in review — no lock cycle appears in the code. Generalize: any blocking operation (queue op, `join`, network call) under a lock needs a comment justifying it; the default answer is no.
- **Granularity = one lock per named invariant, and write the invariant in a comment next to the lock.** Reviewers reflexively check contention (too-coarse) but miss invariant span: lock-per-field tears invariants spanning two fields between acquisitions. If you can't state what a lock guards, it's decorative.
- **Never call unknown code while holding a lock** — callbacks, listeners, logging that might flush, `__eq__`/`hashCode` on user-supplied types, signal emission. Snapshot state under the lock, release, invoke outside.
- **Guarding writes but not reads.** "It's just a read" is not a memory-model argument: readers see mid-rebalance structures (dict resizing, tree rotating) and, without an acquire edge, possibly stale values forever. The lock or snapshot-publication must cover readers too.
- **Every unbounded queue is a lie about capacity.** Producer faster than consumer = OOM at peak load. Decide full-behavior explicitly — block, shed, or coalesce — don't let the allocator decide.
- **Bulkheads per external dependency.** One pool shared by "call slow service X" and "serve all requests" lets X's brownout consume every thread — pool exhaustion is the classic cascading-failure mechanism. Dedicated bounded pool + bounded queue + timeout per dependency.
- **Every blocking call carries a timeout** and a defined behavior on expiry (retry, shed, escalate). Timeout-free `acquire`/`get`/`join`/read is a hang waiting for its trigger.
- **Never test concurrency with sleeps.** `time.sleep(0.1)` "to let the other thread run" flakes in CI forever. Force interleavings with `threading.Event`/barriers/latches, stress-loop the race body thousands of times, and run detectors: `go test -race` (non-negotiable in Go), TSan for C/C++, `loom` for Rust lock-free logic, asyncio debug mode for Python.
- **Shutdown is a concurrency protocol of its own:** stop accepting work → drain or persist queued work → cancel in-flight in dependency order (consumers before their resources) → join with timeouts → close connections. Ad-hoc shutdown (kill, daemon threads evaporating) is where "rare" data loss actually lives. Test it like the hot path.
- **Fairness is a requirement, not a default.** Writer-preferring RW locks starve readers (and vice versa); constant high-priority arrivals starve the low tier; wakeups aren't FIFO. If a consumer *must* eventually run, verify the primitive documents fairness or add aging/quotas.
- **Sharing non-thread-safe clients:** one `sqlite3` connection or SQLAlchemy `Session` across threads/tasks corrupts only under load. One client per thread/task, or an explicitly thread-safe pool — check the library's documented thread-safety.
- **`ThreadPoolExecutor.submit` stores the exception until `.result()`** — a Future nobody reads dies silently. Every spawned unit needs a place its exception lands: TaskGroup/nursery, done-callback that logs, or a join that inspects results.
- **At-least-once and unordered is the default assumption.** Handlers idempotent and order-tolerant unless the specific mechanism documents otherwise.

## Exact numbers, flags, and detection

- `asyncio.run(main(), debug=True)` logs any callback over **100 ms**; run a loop-lag watchdog metric in production — single-request tests never observe loop starvation, only concurrent load does. The diagnostic tell for a blocked loop: latency degrades *globally* (unrelated endpoints, health checks) in multiples of one operation's duration.
- `-W error::RuntimeWarning` in tests turns Python's "coroutine was never awaited" warning into a failure.
- Hung process: **take dumps before restarting** — `py-spy dump --pid` (Python, no restart needed), `jstack`/`kill -3` (JVM, flags deadlocks itself), `SIGQUIT` (Go), `gdb -p` + `thread apply all bt` (native). The evidence dies with the restart.
- Pool math: threads ≈ cores × (1 + wait/compute); an absurd output (thousands) is the signal to go async, not spawn. Little's law first: in-flight = rate × latency — run this arithmetic before tuning anything.
- Uncontended mutex ≈ tens of ns. *Contention*, not locking, is the cost: shard per-thread and aggregate on read (`LongAdder` pattern) instead of hammering one atomic; under real contention a CAS retry loop can burn more CPU than a well-held mutex.
- Free-threaded (no-GIL) CPython builds remove even bytecode-level accidental atomicity — code that "worked" by leaning on the GIL breaks there. Never rely on it.
- `multiprocessing` in any process that also uses threads: force `spawn` (or `forkserver`) start method; `fork` copies locks held by other threads as permanently locked.

## Model selection (compressed)

| Workload | Model |
|---|---|
| Many concurrent I/O waits | async/await or goroutines — OS threads cap ~thousands (MB stacks) |
| CPU-bound Python | processes (`ProcessPoolExecutor`; chunk work so pickling/IPC amortizes) or GIL-releasing native code |
| CPU-bound Go/Rust/Java/C# | pool ≈ core count |
| Pipelines, fan-out/fan-in, cancellation trees | channels/CSP or structured concurrency (`TaskGroup`, nurseries) — the deadlock surface is the channel graph, which you can draw |
| Stateful entities with independent lifecycles | actors: one mailbox, one owner per entity |
| Read-mostly shared config | immutable snapshot behind one atomic reference — beats RWLock on speed and correctness |
| Shared hot mutable state | locks, or per-thread sharding aggregated on read |

Channels vs shared memory: channels when data *flows through stages with ownership transfer*; shared memory + locks when many parties need random access to one structure. Forcing random access through channels reimplements a lock, slowly, with deadlocks.

## Memory-model one-liners

- C/C++: any data race = license to miscompile, not "stale value". `seq_cst` default; `acquire`/`release` only with a proven need and a comment; `relaxed` only for counters you never branch on.
- Java: `volatile` = visibility + ordering edge, never compound atomicity; only `final` fields survive unsafe publication.
- Go: races may corrupt memory; `-race` in CI non-negotiable; "it's just an int" is not an excuse.
- Rust: `Send`/`Sync` kill data races at compile time but not race conditions — check-then-act across two `Mutex` acquisitions still races.
- Python: GIL prevents torn single-object reads, never compound-op races; multiprocessing "shared" state is silently copies.
- JS/TS: no data races, but every `await` is an interleaving point — every fact established before an `await` must be re-validated after it, or the critical section must contain no awaits.

## Self-check before presenting concurrent code

- Every shared mutable datum: name its owner or the lock/edge that guards it — *including all readers*.
- Every check-then-act / read-modify-write: point to the mechanism making the compound op atomic (lock span, CAS, `setdefault`, DB constraint, `O_EXCL`).
- Locks: global acquisition order stated? Anything awaited, blocking, I/O-bound, or user-callable held under one?
- Async: zero blocking calls inside `async def`? Every task owned with an exception sink? Cleanup in `finally`/`async with` (cleanup that awaits can be re-cancelled — shield it)? `CancelledError` re-raised if intercepted?
- Every queue bounded with a stated full-behavior? Every blocking call timeboxed?
- Pool sizes justified by arithmetic, with bulkheads per dependency?
- Run under a race detector and stress loop with forced interleavings — not once, green, on a warm laptop?
- Shutdown path specified (drain order, cancellation order, join timeouts) and exercised by a test?
- Every fact carried across an `await` re-validated or protected?

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 13 baseline (cut/compressed), 0 partial, 0 delta.
- Opus cold-answered everything at full specificity: race taxonomy, GIL counter, Goetz formula (exact 80-thread result), bcrypt-releases-GIL fix, weak-ref task GC + TaskGroup, CancelledError/BaseException, per-language DCL forms, fork+held-locks (knew the 3.14 start-method change), dump-before-restart tooling, snapshot-vs-RWLock, JS await races, Little's law, lock-free vs wait-free with ABA/reclamation.
- Restructured into a correction sheet: residual value is the unprobed operational layer — blocking-put-under-lock deadlock, lock-per-invariant commenting, bulkheads, shutdown protocol, fairness, sleep-free testing, and exact detection flags (asyncio 100 ms debug threshold, `-W error::RuntimeWarning`, loom).
- Worked examples deleted: all three matched probes Opus nailed cold.
