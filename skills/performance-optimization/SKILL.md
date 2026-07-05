---
name: performance-optimization
description: Load when making code or systems faster — diagnosing slowness, profiling, reducing latency or memory, writing or reviewing benchmarks, capacity/latency budgeting, or evaluating optimization proposals. Also load when someone claims "X is slow" or "Y made it faster" and the claim needs to be verified or quantified.
---

# Performance Optimization

You already know the theory cold (Amdahl arithmetic, the Jeff Dean latency table, coordinated omission, flame-graph reading, queueing knees, N+1 batching, Python vectorization). The failure mode in practice is never "didn't know" — it's "knew and didn't run the check." This sheet is the execution discipline plus the few anchors that get dropped.

## Non-negotiable sequence (enforce, don't just recall)

1. **Record the baseline before the first change**: exact command, dataset, environment, numbers *with spread*. No baseline → the "after" claim is unfalsifiable and no regression test is possible.
2. **Triage system-level before opening source.** USE sweep (per-core CPU, swap si/so, `iostat` %util/await, retransmits, pool exhaustion) plus the trace-level question "client, network, or server?". Code-profiling a swapping machine measures the swapping; optimizing a server that's fast in its own logs is a category error.
3. **Run the Amdahl ceiling 1/(1−p) before starting**, and write the stop condition numerically ("p99 < 200ms at 2× load") before writing code. Without a stop condition, optimization continues past usefulness on momentum.
4. **Attack order: delete work → batch/amortize → make faster.** Before profiling anything, grep for loops containing awaits/queries — the N+1 shape (per-item round trips: DB, HTTP, RPC, unbuffered syscalls) explains a plurality of real "it's slow" tickets, and its floor is items × RTT regardless of backend speed.
5. **Re-profile after every landed win** (yesterday's p is stale) and **re-measure end-to-end**, not the microbenchmark: a "faster" function can regress the whole via i-cache bloat, footprint that evicts hotter data, or longer lock hold time.
6. **Land a regression guard** (CI benchmark, dashboard alert on the metric) or the win dies in next quarter's refactor.

## Sanity anchors beyond the standard latency table

| Operation | Rough cost | Back-of-envelope veto it enables |
|---|---|---|
| Predicted branch / function call | ~1ns | — |
| Branch mispredict | ~5–15ns | — |
| Uncontended mutex | ~20–50ns | Uncontended lock in a loop is noise; a *contended* one is a different regime (queueing), not a bigger constant |
| Small malloc / GC alloc | ~50–200ns | GC cost is caused at allocation sites — profile allocations, not CPU, for GC pressure |
| Syscall (post-mitigations) | ~100s ns–1µs | Per-item syscalls in a loop → buffer |
| Thread context switch | ~1–10µs | — |
| Parsing JSON | ~1–10ms per MB | JSON as an internal hot-path format costs ms/MB no matter how good the parser |
| TLS handshake (uncached) | multiple RTTs + ~ms CPU | No keep-alive/pooling = handshake per request |
| Process fork/spawn | ~ms | Per-request fork caps you at hundreds of req/s/core |

If a measured number contradicts this table ("2ms spinning-disk random read", "50µs cross-region call"), the *measurement* is wrong, not the system fast.

## Correction sheet — checks that get skipped

**Measurement validity**
- Debug build check first: `-O0`, cargo without `--release`, coverage instrumentation, `-Xint` → numbers 2–50× off that also rank hot spots *differently*.
- Sampling profilers to rank; tracing profilers for paths/counts only — instrumentation can distort hot loops 10×+ and reorder the ranking (cheap-but-frequent functions inflate most).
- Monotonic clock for durations (`time.perf_counter()`); `time.time()` steps under NTP.
- Interleave A/B runs (never all-A-then-all-B); declare cache-warm vs cache-cold as an explicit condition.
- A delta inside run-to-run variance is a coin flip, not a win — require it to clear the measured spread (JMH/pytest-benchmark report bounds; use them).
- DCE tell: benchmark gets *faster* when you add work, or time doesn't scale with N. Consume results (`Blackhole`, `black_box`); feed non-constant inputs.
- Any latency-under-load figure from a closed-loop generator is invalid — demand fixed arrival rate (wrk2 `--rate`, k6 constant-arrival-rate) with omission-corrected histograms.
- Load-testing at 1/100 scale misses everything superlinear (the O(n²), the index that no longer fits RAM, the lock that only contends at real concurrency). State the scale next to every result.

**Fix-to-cause matching**
- Match fix to percentile: p50 problems are hot-path cost; p99 problems are queueing, GC, locks, cold caches, retries. Trace the *slow requests specifically* — if they share a trigger, optimize the trigger, not the hot path. A change that improves the mean but fattens p99 is usually a regression.
- Caching is not a reflex fix: before adding one, check if the operation can be made cheap or batched; after, monitor hit rate (30% hit rate = complexity with no payoff) and handle stampedes (TTL jitter + single-flight).
- Throughput and latency trade against each other: batching/buffers/concurrency raise throughput and fatten latency. Latency-sensitive capacity: plan ~60–75% utilization at peak — wait time grows hyperbolically toward saturation.
- Adding threads made it *slower* → suspect false sharing (two threads writing the same 64B cache line); pad or separate per-thread data.
- Flame graphs: act only on wide *self-time* leaves (or avoidable subtrees) backed by hundreds of samples; width is inclusive, so a wide frame with wide children is just a caller.

**When not to optimize**
- Cost below the system's noise floor; simple version not shipped yet (optimizing hypothetical load with real complexity); win requires breaking a correctness invariant (get explicit sign-off, never trade silently); bottleneck is organizational (negotiate the constraint instead).

## Self-check before presenting conclusions

- Every claim anchored to a named measurement (tool, workload, before/after with spread)?
- Amdahl arithmetic consistent — claimed end-to-end speedup vs component's measured share?
- Numbers pass the latency-table physical-sense check?
- Benchmarks: warmup, results consumed, non-constant inputs, open-loop load, variance reported, A/B interleaved?
- Tails reported separately from mean; fix matched to the right percentile's cause?
- Regression guard in place?

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 12 baseline (cut/compressed), 0 partial, 0 delta.
- Opus cold reproduced the entire prior skill: the Amdahl/N+1 triage worked example (rejecting the Rust-serializer trap, adding orjson), Jeff Dean numbers with conversion ratios, a quantitatively correct 1000× coordinated-omission example, the 1−0.99¹⁰⁰≈63% fan-out math, queueing knee and utilization targets, flame-graph misreads, Python loop rankings, batching floors, and a full USE sweep — several answers exceeded the skill (hedged requests, tail-based trace sampling, x-axis-not-time-ordered).
- No knowledge gap found. Restructured from survey to execution checklist: the residual value is forcing the discipline (baseline-first, Amdahl ceiling + stop condition, end-to-end re-measure, open-loop load, regression guard) that the model knows when quizzed but skips when working, plus the unprobed extended op-cost table used for back-of-envelope vetoes.
