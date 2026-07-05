---
name: go-development
description: Loads expert Go judgment for goroutine lifecycle management, channels vs mutexes, context discipline, interface and error design, slice/nil gotchas, generics restraint, and pprof-driven performance work. Use when writing or reviewing Go services, debugging goroutine leaks or races, designing packages and interfaces, or structuring Go tests and project layout.
---

# Go Development

## Core mental model (anchors)

1. Before every `go` statement answer two questions: how does this goroutine terminate, and who waits for it? No answers, no `go`. Goroutine leaks are the #1 Go production bug class.
2. Channels model movement (pipelines, hand-off, completion); mutexes model state (caches, counters). Both migration directions are legitimate — a mutex design growing "wait until condition" logic wants channels; a channel design growing request/reply pairs wants a mutex.
3. Interfaces are discovered by consumers: defined in the package that *uses* them, sized to the methods it calls. Accept interfaces, return concrete types.
4. Errors: `%w` chains, `errors.Is`/`As`, handled exactly once (log or return, not both).

## Current state (verified July 2026)

- **Go 1.26 is current** (Feb 2026). The facts cold answers hedge or miss:
  - **Green Tea GC is on by default in 1.26** (was `GOEXPERIMENT=greenteagc` in 1.25) — roughly 10–40% GC-overhead reduction in GC-heavy programs, no code changes. Re-measure old allocation-shaving folklore before porting it.
  - Container-aware `GOMAXPROCS` since 1.25 — `uber-go/automaxprocs` is obsolete on modern versions.
  - `testing/synctest` finalized in 1.25 (virtualized clock) — use it instead of `time.Sleep` in timeout/retry tests.
  - `encoding/json/v2` exists behind `GOEXPERIMENT=jsonv2` (1.25+) — mention as future, don't default to it.
  - Per-iteration loop vars and `for range n` over ints since 1.22.
- Layout: `cmd/<binary>/`, `internal/` (compiler-enforced), flat root for small libraries; no reflexive `pkg/`, no `utils`/`common`/`models` grab-bags — packages organized by capability.

## Judgment calls where reflexes differ

- **Fan-in + early return = orphaned senders.** When a collector can return early (ctx cancellation) while workers send to an unbuffered results channel, the one-line idiomatic fix for *known cardinality* is `make(chan result, n)` — buffer to the number of sends so every sender can complete. The cold-answer reflex calls buffering a band-aid and prescribes drain-until-closed or cancellable sends; those are right for long-lived/unbounded senders, but for bounded fan-in the buffer IS the fix — ship it today, schedule the errgroup rewrite.
- **Unmeasured `RWMutex` is premature** and can be slower than `Mutex`; buffer sizes need a stateable reason (burst size, semaphore `make(chan struct{}, N)`, fan-in cardinality) — "buffered so sends don't block" hides deadlocks and loses backpressure.
- **Generics litmus:** type parameter appearing once in the signature and never returned → interface parameter instead. If the pre-1.18 answer was "write it twice," that often remains the answer. Calibrate "worth it" against the stdlib `slices`/`maps`/`cmp` packages.
- Process root-causes over code fixes: a race that shipped means `-race` wasn't in CI — `go test -race ./...` on every merge is the fix; the code bug is the symptom.

## Diagnosis priors (compressed)

- Goroutine leak: diff `/debug/pprof/goroutine?debug=1` twice, 10 min apart. Base rates: parked on chan send (reader bailed) > chan receive (never-closed) > network I/O without deadlines > ticker loops with no exit.
- "Concurrent map writes" despite "has a mutex": `go test -race -count=50`; the classic is a *write under RLock* in a read-through cache. Fixes ranked: full write lock (load cheap) → singleflight (load does I/O) → double-checked locking; `sync.Map` is wrong for read-modify-write (its two blessed workloads: write-once/read-many; disjoint key sets).
- Performance: `-benchmem` allocs/op before ns/op; wins by base rate: unpreallocated slices/maps → `[]byte`↔`string` churn → `fmt` in hot paths → interface boxing → JSON cost.

## Pitfalls checklist (one-liners — cold answers reproduce the mechanisms)

Typed-nil interface (return literal `nil`, never a possibly-nil concrete pointer); slice append aliasing (full-slice expression `a[low:high:max]`, `slices.Clone` before callee append, never retain a small sub-slice of a huge buffer — pins the array); `defer` in loops (extract function); missing `case <-ctx.Done()` makes cancellation advisory; `defer cancel()` after `WithTimeout`; `context.WithValue` for cross-cutting metadata only; `%v` severs error chains — `%w`; package sentinels over ad-hoc `errors.New`; `wg.Add(1)` before `go`; map iteration order randomized; `time.NewTicker` + `defer Stop()` (1.23+ GC collects unreferenced tickers, but the goroutine looping around one still leaks without an exit); handler background work needs `context.WithoutCancel` (1.21+) *and* registration with shutdown wait; interface-per-struct "for mockability" is pollution — mock at architectural boundaries only; `v := v` is dead on 1.22+ (check go.mod before deleting from old code).

Expanded — the two that still ship in 2026 codebases:

- **`http.Server` zero-value timeouts are infinite**: set `ReadHeaderTimeout` (slowloris), `ReadTimeout`, `WriteTimeout`, `IdleTimeout` (keep-alive reaping); and `http.DefaultClient` has NO timeout — construct `&http.Client{Timeout: ...}` or use per-request contexts. Pair with the shutdown shape: `signal.NotifyContext` → `srv.Shutdown(ctxWithTimeout)` drains in-flight.
- **Worker pool exit plan** — the shape to memorize: `errgroup.WithContext`; each worker selects on `ctx.Done()` (exit 1: cancellation) and `jobs` channel-closed (exit 2: producer done); an error fails the group and cancels siblings; `g.Wait()` (exit 3: someone provably waits).

## Testing conventions (compressed)

Named cases + `t.Run` + `t.Parallel()`; `cmp.Diff` for structs; test the public API from `package foo_test`; `t.Helper()` in assertion helpers; `synctest` for anything clock-driven on 1.25+; `go vet ./...` always (catches WaitGroup copying and more).

## Verification and stopping rule

1. Point to every goroutine's exit plan and waiter.
2. Assume `go vet` and `go test -race ./...` run — survives both?
3. Check every `append` on shared/returned slices for aliasing; every interface-typed return for typed nil.
4. Version-gate: loop vars/range-int 1.22+, `WithoutCancel` 1.21+, synctest 1.25+, Green-Tea-default claims 1.26.
5. Performance assertions name the profile or benchmark that would confirm them.

Done when the code reads top-to-bottom without goroutine bookkeeping in your head, interfaces have ≤3 methods in consumer packages, and pprof puts business logic on top. Go punishes cleverness — when in doubt, write the dumber version and stop.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta.
- Opus cold nails: goroutine-leak triage, typed nil, slice aliasing + three-index defense, ticker GC change in 1.23, RLock-write race + singleflight/sync.Map judgment, server timeouts, WithoutCancel, synctest, layout/pkg consensus. Skill restructured to a correction sheet.
- Remaining value: Green Tea GC *default-on in 1.26* (cold answers hedge "moving toward default"), the buffer-to-cardinality fan-in judgment call (cold reflex dismisses it as a band-aid), jsonv2 experiment status, and the compressed base-rate orderings.
