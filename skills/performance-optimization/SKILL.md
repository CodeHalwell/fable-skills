---
name: performance-optimization
description: Load when making code or systems faster — diagnosing slowness, profiling, reducing latency or memory, writing or reviewing benchmarks, capacity/latency budgeting, or evaluating optimization proposals. Also load when someone claims "X is slow" or "Y made it faster" and the claim needs to be verified or quantified.
---

# Performance Optimization

## Core mental model

- **Measure first; the bottleneck is never where intuition says.** Decades of profiling folklore agree: even experts guess the hot spot wrong most of the time. Any optimization applied before profiling has negative expected value — it adds complexity with unknown (often zero, sometimes negative) speedup. The profile is not a formality; it is the diagnosis.
- **Amdahl's law is arithmetic you must actually run.** Speedup = 1 / ((1−p) + p/s), where p is the fraction of total time in the part you're optimizing and s is its speedup. Optimizing 20% of runtime by 10x yields 1/(0.8+0.02) ≈ 1.22x overall. Corollary: eliminating a 5% cost caps at 1.05x no matter how heroic the work — do this arithmetic *before* starting, and stop when p of what's left is small.
- **Know the latency hierarchy in orders of magnitude** — it decides architecture before any profiler runs: L1 cache ~1ns, main memory ~100ns (≈100x L1), SSD random read ~100µs (≈1000x RAM), same-datacenter network round trip ~500µs, spinning disk seek ~10ms, cross-continental round trip ~100ms+. Practical conversions: one network hop ≈ a million L1 hits; one disk seek ≈ 100k RAM reads. If a design does N sequential network calls, its floor is N × RTT and nothing in-process matters until N shrinks.
- **Order of attack: do less work, then do it in bulk, then do it faster.** (1) Algorithmic/structural — better complexity class, caching, skipping work entirely, precomputation. (2) Batching/amortization — fewer syscalls, fewer round trips, fewer allocations, vectorized ops. (3) Micro — locality, branch predictability, SIMD, hand-tuning. A 10-line O(n²)→O(n log n) fix beats a month of category-3 work; category 3 before 1 is malpractice.
- **Mean latency is a vanity metric; tails are the product.** Users experience p99 (and with fan-out, worse: hit 100 backends per request, each 99th-percentile-slow 1% of the time → ~63% of requests see at least one slow backend). Optimize and report p50/p95/p99 separately; a change that improves mean but fattens p99 (e.g., adding a cache with a slow miss path plus eviction storms) is usually a regression.

## Decision frameworks

### Choosing the measurement tool
| Symptom | Tool/approach |
|---|---|
| "Whole service is slow" | Distributed trace / flame graph of a real request first — find *which* component before profiling inside one |
| CPU-bound process | Sampling profiler in production or prod-like load: `py-spy` (Python, attach to live proc), `perf` + flamegraph (native/JVM via async-profiler), `pprof` (Go). Sampling ≈ negligible overhead, no code change |
| Slow but CPU idle | It's waiting: off-CPU analysis. Check I/O waits, lock contention, connection-pool exhaustion, GC pauses. `py-spy dump` / thread dumps show where threads sit blocked |
| Memory growth / GC pressure | Allocation profiler (`tracemalloc`, Go heap profile, async-profiler alloc mode), not a CPU profiler — GC cost is caused at allocation sites |
| One function, micro-level | Proper benchmark harness: `pytest-benchmark`/`timeit` (Python), JMH (JVM), `criterion` (Rust), `go test -bench` with `b.N`. Never a wall-clock print around one call |
| DB query slow | `EXPLAIN ANALYZE` before touching indexes; check rows-scanned vs rows-returned ratio |

- Profile the *representative* workload: production data shapes, warm caches (or cold, if that's the real case), realistic concurrency. Profiling a dev laptop against toy data finds toy bottlenecks.

### Is this optimization worth doing? (run before writing code)
1. From the profile, get p = fraction of end-to-end cost in the target. Compute the Amdahl ceiling assuming *infinite* speedup: 1/(1−p). If the ceiling doesn't meet the goal, this target alone is insufficient — find a bigger p or stack multiple wins.
2. Estimate s realistically (removing a network hop: use the RTT numbers; algorithmic: use n and the complexity delta; micro: 1.2–3x is typical, 10x is rare).
3. Weigh against the complexity tax: an optimization that obscures code needs roughly (readers-per-year × comprehension cost) < (users × time saved). Hot inner loops earn ugliness; cold paths never do.
4. Define the stop condition numerically before starting ("p99 < 200ms at 2x current load") — otherwise optimization continues past usefulness by momentum.

### Memory locality and allocation (category-2/3 wins that actually matter)
- Cache lines are 64 bytes: arrays of structs traversed field-wise waste ~90% of each line; struct-of-arrays or columnar layouts (NumPy, Arrow, ECS) turn memory-bound scans into streaming ones. Sequential access lets the prefetcher hide the 100ns; pointer-chasing (linked lists, hash-map hopscotch, object graphs) eats the full miss each hop — a linked list can be 10–100x slower to traverse than a vector of the same data.
- Allocation pressure: in GC languages the cost isn't `new`, it's the collector work proportional to allocation rate, surfacing as CPU tax and pause-time tails. Fixes in order: don't allocate in the hot loop (hoist, reuse buffers), batch small objects into arrays of primitives, then object pools (last resort — pools resurrect use-after-free bug classes).
- In Python specifically: the biggest wins are almost never Python-level micro-opts. Move loops into C-backed bulk ops (NumPy vectorization, `str.join`, comprehensions over `+=` on strings/lists, `set` membership over list scans), or move the hot kernel to a compiled path. Rewriting `for` as `while` is noise.

### Benchmark design (getting numbers that aren't lies)
- **Warmup:** JIT runtimes (JVM, JS, PyPy, .NET) run interpreted/tiered code for thousands of iterations before steady state. Report steady-state only; JMH/criterion handle this — hand-rolled loops don't.
- **Dead-code elimination:** a benchmark whose result is unused gets optimized away; you're timing an empty loop. Consume every result (JMH `Blackhole`, `criterion::black_box`, accumulate-and-print). A benchmark that gets *faster* when you add work is DCE'd.
- **Constant folding:** benchmarking with compile-time-known inputs lets the optimizer precompute the answer. Feed inputs from outside the compilation unit / from `black_box`.
- **Coordinated omission:** closed-loop load generators wait for each response before sending the next, so when the system stalls, the generator politely stops measuring — hiding exactly the tail you care about. Use open-loop, fixed-arrival-rate generation and correct-for-omission histograms (wrk2, HdrHistogram-based tools). Any latency-under-load number from a closed-loop tool overstates tail performance, often by orders of magnitude.
- Control the environment: pin CPU frequency scaling (or at least report it), watch thermal throttling on laptops, isolate from noisy neighbors, run enough iterations to report variance — a lone mean without spread is not a measurement. Compare distributions, not single runs.

## Failure modes & pitfalls

- **Optimizing without a baseline number.** No "before" measurement means the "after" claim is unfalsifiable and the regression test impossible. Record the exact command, dataset, and numbers before the first change.
- **Trusting the microbenchmark over the macro effect.** Function-level win, end-to-end nothing (Amdahl), or end-to-end *regression*: the "faster" version bloats instruction cache, adds memory footprint that evicts hotter data, or holds a lock longer. Always confirm at the end-to-end level after landing.
- **Caching as the reflex fix.** A cache added without hit-rate measurement is a bug generator: invalidation bugs, stampedes on expiry (add jitter + single-flight/request coalescing), unbounded memory, stale reads, and a fat miss-path tail. Before caching, check whether the underlying op can just be made cheap or batched. After caching, monitor hit rate — a 30% hit rate cache is complexity with no payoff.
- **N+1 everywhere, not just ORMs.** The pattern — per-item round trips inside a loop — appears with DB queries, HTTP calls, RPCs, and even syscalls (per-line unbuffered writes). Latency floor = items × RTT. Fix by batching (`SELECT ... WHERE id IN (...)`, bulk endpoints, pipelining, buffered I/O). This single pattern explains a plurality of real-world "it's slow" tickets; grep for loops containing awaits/queries first.
- **Confusing throughput and latency goals.** Batching, pipelining, and bigger buffers raise throughput but add latency; concurrency raises throughput until queueing explodes latency. Per queueing theory, latency grows nonlinearly as utilization → 1: running a server at 90%+ utilization *guarantees* terrible tails; capacity-plan for ~60–75% at peak if latency matters.
- **`time.time()` around one call, once.** Single-shot timings measure cache state, page faults, and scheduler luck. Also: `time.time()` can go backwards (NTP); use `time.perf_counter()`/monotonic clocks for durations, always.
- **Benchmarking the allocator/pagecache instead of your code.** First run touches cold file cache and faults pages in; comparing run 1 of A against run 5 of B is meaningless. Interleave A/B runs; state cache-warm vs cache-cold as an explicit condition.
- **Premature `__slots__`/inlining/bit-tricks while an O(n²) sits in the profile.** The tell: optimization effort allocated by what's fun rather than by the flame graph's widest frame. Discipline: sort by width in the profile, attack top-down, re-profile after each change (the bottleneck *moves*).
- **Reading the flame graph wrong.** Width = inclusive time (self + children); a wide frame with wide children is just a caller — the actionable frames are wide *self*-time leaves, or wide frames you can avoid calling entirely. Also check the sample count is large enough that widths are stable (hundreds+ of samples in frames you act on).
- **Ignoring variance and calling a 3% delta a win.** If run-to-run noise is ±5%, a 3% improvement is a coin flip. Report the spread; require the improvement to clear the noise floor (or use a proper statistical comparison, e.g., pytest-benchmark's, JMH's error bounds) before claiming victory.

## Worked micro-examples

**1. Amdahl triage of a 800ms API endpoint.** Trace shows: 450ms in 15 sequential DB queries (N+1 over 15 items × ~30ms each), 250ms in JSON serialization of a 5MB payload, 100ms other. Proposal on the table: "rewrite serializer in Rust, ~10x faster."
- Serializer: p = 250/800 = 0.31, s = 10 → speedup = 1/(0.69 + 0.031) = 1.39x → 577ms. Weeks of work.
- Batch the queries: 15 round trips → 1 `WHERE id IN (...)`: 450ms → ~35ms. New total ≈ 385ms, a 2.1x, one afternoon. Then the serializer is p = 0.65 of the remainder — *now* it's the right target, but check first whether the 5MB payload should exist at all (field selection/pagination might delete the cost instead of optimizing it). Expert order: shrink work → batch → only then make faster.

**2. Batching arithmetic from the latency table.** Writing 10,000 rows via single-row INSERTs, DB RTT 1ms: floor = 10,000 × 1ms = 10s regardless of DB speed. One batched multi-row insert (or `COPY`): 1 RTT + server-side cost ≈ tens of ms. The 100–1000x is pure round-trip elimination — predictable from the hierarchy *before* measuring, which is what the hierarchy is for. Same math kills chatty microservice call chains: 20 sequential internal RPCs × 0.5ms = 10ms latency floor, before any work happens.

**3. Locality in Python, concretely.**
```python
# Sum one field over 10M "points"
pts = [{"x": random(), "y": random()} for _ in range(10_000_000)]
s = sum(p["x"] for p in pts)          # pointer-chasing: dict per element, ~seconds
xs = np.random.rand(10_000_000)        # columnar: contiguous float64 buffer
s = xs.sum()                           # streams 80MB at memory bandwidth, ~10ms
```
The ~100x is locality + eliminating per-element interpreter work: one contiguous 80MB scan vs 10M scattered heap objects with per-item dict lookups and refcounting. The design lesson generalizes: choose the data layout (columnar, arrays-of-primitives) before micro-tuning access to a bad layout.

## Self-check before presenting performance conclusions

- Is every claim anchored to a measurement I can name (tool, workload, dataset, before/after numbers with spread)? "Should be faster" is not a result.
- Did I run the Amdahl arithmetic — is the claimed end-to-end speedup consistent with the component's measured share of total time?
- Sanity-check against the latency hierarchy: does the number make physical sense? (A "2ms disk-bound random-read op" or a "50µs cross-region call" means the measurement is wrong, not the system fast.)
- For benchmarks: warmup handled, results consumed (no DCE), inputs non-constant, open-loop load for latency-under-load claims, variance reported, A/B interleaved?
- Did I report tail percentiles, not just mean — and check the change didn't trade a better mean for a worse p99?
- Is there a regression guard (benchmark in CI, dashboard alert) so the win survives next quarter's refactor?
