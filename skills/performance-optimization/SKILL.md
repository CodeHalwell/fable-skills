---
name: performance-optimization
description: Load when making code or systems faster — diagnosing slowness, profiling, reducing latency or memory, writing or reviewing benchmarks, capacity/latency budgeting, or evaluating optimization proposals. Also load when someone claims "X is slow" or "Y made it faster" and the claim needs to be verified or quantified.
---

# Performance Optimization

## Core mental model

- **Measure first; the bottleneck is never where intuition says.** Even experts guess the hot spot wrong most of the time — this is the most replicated finding in performance folklore. Any optimization applied before profiling has negative expected value: it adds complexity with unknown (often zero, sometimes negative) speedup. The profile is the diagnosis, not a formality.
- **Amdahl's law is arithmetic you must actually run.** Speedup = 1 / ((1−p) + p/s), where p = fraction of total time in the part you're optimizing, s = its speedup factor. Optimizing 20% of runtime by 10x yields 1/(0.8 + 0.02) ≈ 1.22x overall. Corollaries:
  - Eliminating a 5% cost entirely caps the win at 1.05x, no matter how heroic the work.
  - Run the ceiling check (s → ∞ gives 1/(1−p)) *before* starting; if the ceiling doesn't meet the goal, find a bigger p.
  - After each win the bottleneck moves — re-profile; yesterday's p is stale.
- **Know the latency hierarchy in orders of magnitude** — it decides architecture before any profiler runs: L1 cache ~1ns; main memory ~100ns (≈100× L1); SSD random read ~100µs (≈1000× RAM); same-datacenter round trip ~500µs; spinning-disk seek ~10ms; cross-continental round trip ~100ms+. Practical conversions: one network hop ≈ a million L1 hits; one disk seek ≈ 100k RAM reads. If a design makes N sequential network calls, its latency floor is N × RTT, and nothing in-process matters until N shrinks.
- **Order of attack: do less work, then do it in bulk, then do it faster.**
  1. Algorithmic/structural — better complexity class, caching, skipping work entirely, precomputation, not fetching data you don't use.
  2. Batching/amortization — fewer round trips, fewer syscalls, fewer allocations, vectorized bulk operations.
  3. Micro — locality, branch predictability, SIMD, hand-tuning.
  A 10-line O(n²)→O(n log n) fix beats a month of category 3. Doing category 3 while a category 1 problem sits in the profile is malpractice.
- **Mean latency is a vanity metric; tails are the product.** Users experience p99, and fan-out makes tails compound: touch 100 backends per request, each slow (p99) 1% of the time, and ~63% of requests (1 − 0.99¹⁰⁰) hit at least one slow backend. Report p50/p95/p99 separately; a change that improves the mean but fattens p99 is usually a regression.

## Decision frameworks

### Choosing the measurement tool
| Symptom | Tool / approach |
|---|---|
| "The whole service is slow" | Distributed trace / flame graph of a real request first — identify *which* component before profiling inside one |
| CPU-bound process | Sampling profiler on production or prod-like load: `py-spy` (Python; attaches to a live process), `perf` + flamegraphs or async-profiler (native/JVM), `pprof` (Go). Sampling ≈ negligible overhead, no code changes |
| Slow but CPU mostly idle | It's waiting — do off-CPU analysis: I/O waits, lock contention, connection-pool exhaustion, GC pauses. `py-spy dump` / `jstack` thread dumps show where threads sit blocked |
| Memory growth / GC pressure | Allocation profiler (`tracemalloc`, Go heap profile, async-profiler alloc mode) — not a CPU profiler; GC cost is caused at allocation sites |
| One function, micro level | A real harness: `pytest-benchmark`/`timeit` (Python), JMH (JVM), `criterion` (Rust), `go test -bench`. Never a wall-clock print around a single call |
| Slow DB query | `EXPLAIN ANALYZE` before touching indexes; the rows-scanned vs rows-returned ratio is the smoking gun |
| Latency under load | Open-loop load generator (wrk2-style, fixed arrival rate) + HdrHistogram percentiles — see coordinated omission below |

- Profile the *representative* workload: production data shapes and sizes, realistic concurrency, cache state matching reality (warm or cold — pick and state it). Profiling toy data on a dev laptop finds toy bottlenecks.

### Is this optimization worth doing? (run before writing code)
1. From the profile: p = the target's fraction of end-to-end cost. Compute the Amdahl ceiling 1/(1−p). Ceiling below goal → wrong target.
2. Estimate s realistically: removing a network hop — use the RTT table; algorithmic — use n and the complexity delta; micro-tuning — 1.2–3× is typical, 10× is rare.
3. Weigh the complexity tax: hot inner loops earn ugliness; cold paths never do. An optimization that obscures code must save more (users × time) than it costs (readers × comprehension).
4. Write the stop condition numerically before starting ("p99 < 200ms at 2× current load"). Without it, optimization continues past usefulness on momentum.

### Memory locality and allocation (the category-2/3 wins that actually matter)
- **Cache lines are 64 bytes.** Arrays of structs traversed one-field-at-a-time waste most of every line fetched; struct-of-arrays / columnar layouts (NumPy, Arrow, ECS) turn scattered reads into streaming ones. Sequential access lets the hardware prefetcher hide the 100ns memory latency; pointer-chasing (linked lists, node-based maps, object graphs) pays the full miss on every hop — traversing a linked list can be 10–100× slower than a contiguous array of the same data.
- **Allocation pressure:** in GC languages the visible cost isn't `new`, it's collector work proportional to allocation rate, surfacing as CPU tax and pause-time tails. Fix in order: don't allocate in the hot loop (hoist, reuse buffers); batch small objects into arrays of primitives; object pools last (pools reintroduce use-after-free bug classes — earn them with a measurement).
- **Python-specific:** the big wins are almost never Python-level micro-edits. Move loops into C-backed bulk operations — NumPy/polars vectorization, `str.join` over `+=` in a loop, `set` membership over list scans, comprehensions over append loops — or move the hot kernel to a compiled path. Rewriting `for` as `while` is noise; changing the data layout is signal.
- **False sharing (multithreaded):** two threads writing different variables that share a cache line serialize on cache-coherence traffic. Symptom: adding threads makes it *slower*. Fix: pad or separate per-thread data to different lines.

### Benchmark design (numbers that aren't lies)
- **Warmup:** JIT runtimes (JVM, JS, .NET, PyPy) run slow-tier code for the first thousands of iterations. Measure steady state only; JMH and criterion handle this — hand-rolled timing loops don't.
- **Dead-code elimination:** a benchmark whose result is unused gets optimized away — you're timing an empty loop. Consume every result (JMH `Blackhole`, Rust `std::hint::black_box`, accumulate-and-print). Diagnostic: a benchmark that gets *faster* when you add work has been DCE'd.
- **Constant folding:** compile-time-known inputs let the optimizer precompute answers. Feed inputs from outside the compilation unit or through `black_box`.
- **Coordinated omission:** closed-loop load generators send the next request only after the previous response — so when the system stalls, the generator politely stops measuring, hiding exactly the tail you care about. Use open-loop, fixed-arrival-rate generation and omission-correcting histograms (wrk2, HdrHistogram-based tools). Any latency-under-load figure from a closed-loop tool overstates tail performance, often by orders of magnitude.
- **Environment control:** pin or report CPU frequency scaling; beware thermal throttling on laptops; isolate from noisy neighbors; interleave A/B runs rather than "all A, then all B" (drift and cache state contaminate); run enough iterations to report variance. A lone mean without spread is not a measurement. Compare distributions, not single runs.
- **Clocks:** use monotonic clocks for durations (`time.perf_counter()` in Python) — `time.time()` can step backwards under NTP.

## Failure modes & pitfalls

- **Optimizing without a baseline number.** No "before" measurement means the "after" claim is unfalsifiable and no regression test is possible. Record the exact command, dataset, environment, and numbers before the first change.
- **Trusting the microbenchmark over the macro effect.** Function-level win, end-to-end nothing (Amdahl) — or end-to-end *regression*: the "faster" version bloats the instruction cache, grows memory footprint that evicts hotter data, or holds a lock longer. Always re-measure end-to-end after landing.
- **Caching as the reflex fix.** A cache added without hit-rate measurement is a bug generator: invalidation bugs, expiry stampedes (fix with TTL jitter + single-flight request coalescing), unbounded memory, stale reads, and a fat miss-path tail. Before caching, check whether the underlying operation can simply be made cheap or batched. After caching, monitor the hit rate — a 30% hit-rate cache is complexity with no payoff.
- **N+1 everywhere, not just ORMs.** The shape — per-item round trips inside a loop — appears with DB queries, HTTP calls, RPCs, and syscalls (per-line unbuffered writes). Latency floor = items × RTT. Fix by batching: `SELECT ... WHERE id IN (...)`, bulk endpoints, pipelining, buffered I/O. This one pattern explains a plurality of real "it's slow" tickets; grep for loops containing awaits/queries before profiling anything.
- **Confusing throughput and latency goals.** Batching, pipelining, and bigger buffers raise throughput but add latency; more concurrency raises throughput until queueing explodes latency. Queueing theory: wait time grows nonlinearly as utilization → 1 — running a latency-sensitive server at 90%+ utilization *guarantees* terrible tails. Capacity-plan for ~60–75% at peak if latency matters.
- **Single-shot timing.** One `time.time()` pair around one call measures cache state, page faults, and scheduler luck, not the code. Iterate, warm up, report spread.
- **Benchmarking the page cache instead of your code.** First run touches cold file cache and faults pages in; comparing run 1 of A against run 5 of B is meaningless. Interleave runs; declare cache-warm vs cache-cold as an explicit experimental condition.
- **Optimizing by what's fun, not by the flame graph.** `__slots__`, bit tricks, and inlining while an O(n²) join sits at 60% of the profile. Discipline: sort frames by width, attack top-down, re-profile after each change.
- **Reading the flame graph wrong.** Width = inclusive time (self + children); a wide frame with wide children is just a caller. Actionable targets are wide *self-time* leaves, or wide subtrees you can avoid invoking entirely. Also confirm sample counts are large enough that the widths you act on are stable (hundreds of samples minimum).
- **Calling a delta inside the noise floor a win.** If run-to-run spread is ±5%, a 3% improvement is a coin flip. Require the delta to clear the measured variance (pytest-benchmark and JMH report error bounds — use them) before claiming victory.
- **Optimizing the wrong percentile.** Shaving p50 while p99 is the SLA breach; or "fixing" tail latency caused by GC/compaction pauses with code tweaks that don't touch the pause source. Match the fix to the percentile: p50 problems are usually hot-path cost; p99 problems are usually queueing, GC, locks, cold caches, or retries.
- **Ignoring the cost of the measurement itself.** Tracing/instrumented profilers can distort hot loops by 10x+ and *reorder* the ranking (cheap functions called often inflate most). For ranking hot spots, prefer sampling profilers; use tracing for call counts and exact paths.

## Worked micro-examples

### 1. Amdahl triage of an 800ms API endpoint
Trace shows: 450ms in 15 sequential DB queries (N+1 over 15 items, ~30ms each), 250ms serializing a 5MB JSON payload, 100ms other. On the table: "rewrite the serializer in Rust, ~10× faster."
- Serializer math: p = 250/800 ≈ 0.31, s = 10 → overall = 1/(0.69 + 0.031) ≈ 1.39× → ~577ms. Weeks of work, misses any sub-500ms goal.
- Batch the queries instead: 15 round trips → 1 `WHERE id IN (...)`: 450ms → ~35ms. New total ≈ 385ms. That's 2.1× for an afternoon.
- Re-profile: serialization is now p ≈ 0.65 of what remains — *now* it's the right target. But apply "do less work" first: does the client use all 5MB? Field selection or pagination may delete the cost rather than optimize it. Expert order: shrink → batch → only then make faster.

### 2. Batching arithmetic straight from the latency table
Writing 10,000 rows via single-row INSERTs at 1ms DB RTT: floor = 10,000 × 1ms = 10s, *regardless of database speed*. One batched multi-row INSERT (or `COPY`): 1 RTT + server-side work ≈ tens of ms. The ~100–1000× is pure round-trip elimination, predictable from the hierarchy before any measurement — that's what the hierarchy is for. The same arithmetic kills chatty microservice chains: 20 sequential internal RPCs × 0.5ms = 10ms latency floor before any actual work happens.

### 3. Locality in Python, concretely
```python
import numpy as np, random
# Sum one field over 10M "points"
pts = [{"x": random.random(), "y": random.random()} for _ in range(10_000_000)]
s = sum(p["x"] for p in pts)      # seconds: 10M scattered heap objects,
                                  # per-item dict lookup + refcounting
xs = np.random.rand(10_000_000)   # one contiguous 80MB float64 buffer
s = xs.sum()                      # ~10ms: streams at memory bandwidth
```
The ~100× gap is locality plus eliminating per-element interpreter work: one sequential 80MB scan versus 10M pointer-chased dicts. The transferable lesson: choose the data *layout* (columnar, arrays of primitives) before micro-tuning access to a bad layout — layout is a category-1 decision wearing category-3 clothes.

## Self-check before presenting performance conclusions

- Is every claim anchored to a named measurement (tool, workload, dataset, before/after numbers with spread)? "Should be faster" is not a result.
- Does the Amdahl arithmetic hold — is the claimed end-to-end speedup consistent with the component's measured share of total time?
- Sanity-check against the latency hierarchy: does each number make physical sense? A "2ms random read from spinning disk" or a "50µs cross-region call" means the *measurement* is wrong, not the system fast.
- For benchmarks: warmup handled; results consumed (no DCE); inputs non-constant; open-loop load for any latency-under-load claim; variance reported; A/B interleaved?
- Tail percentiles reported, not just mean — and confirmed the change didn't trade a better mean for a worse p99?
- Is the fix matched to the right percentile's cause (hot path vs queueing/GC/locks)?
- Is there a regression guard (benchmark in CI, dashboard alert on the metric) so the win survives next quarter's refactor?
