---
name: go-development
description: Loads expert Go judgment for goroutine lifecycle management, channels vs mutexes, context discipline, interface and error design, slice/nil gotchas, generics restraint, and pprof-driven performance work. Use when writing or reviewing Go services, debugging goroutine leaks or races, designing packages/interfaces, or structuring Go tests and project layout.
---

# Go Development

## Core mental model

1. **Every goroutine needs an exit plan before you write `go`.** Goroutine leaks are the #1 production bug class in Go services: a goroutine blocked forever on a channel nobody reads, a `ctx` nobody cancels, or a loop with no stop signal. Before every `go` statement, answer: *how does this goroutine terminate, and who waits for it?* If you can't answer both, don't start it.
2. **Share memory by communicating — but only when there's a flow.** Channels model pipelines, fan-in/fan-out, ownership transfer, and signaling. Mutexes model shared state (caches, counters, config). The slogan is not "always channels"; a mutex around a map is simpler and faster than a channel-guarded map service goroutine.
3. **`context.Context` is the cancellation and deadline bus, and it flows one way.** First parameter of every function that blocks, does I/O, or can be slow: `func F(ctx context.Context, ...)`. Never store a context in a struct (except transitional request wrappers); never pass `nil` (use `context.Background()`/`TODO()`); values on context are for request-scoped cross-cutting data (trace IDs, auth principal), not function parameters in disguise.
4. **Interfaces are discovered by consumers, not designed by producers.** Define the interface in the package that *uses* it, sized to exactly what it calls (often one method). Return concrete types; accept interfaces. A package exporting `type FooInterface` next to `type fooImpl` with a constructor returning the interface is Java smuggled into Go.
5. **Errors are values with a wrapping convention.** `fmt.Errorf("opening config: %w", err)` builds a cause chain; `errors.Is` (sentinel comparison) and `errors.As` (type extraction) walk it. Callers must never string-match error text. Wrap with context at each layer; handle (log OR return, never both) exactly once.

## Current state (verified July 2026)

- **Go 1.26 is current** (Feb 2026; 1.25 still supported — Go supports the last two minors). Notables: 1.25 finalized **`testing/synctest`** (deterministic testing of concurrent code with a fake clock — use it instead of `time.Sleep` in tests) and container-aware `GOMAXPROCS` (respects cgroup CPU limits — the old uber-go/automaxprocs workaround is unnecessary on 1.25+). The **Green Tea GC**, experimental in 1.25, is **on by default in 1.26** (roughly 10–40% GC overhead reduction in GC-heavy programs). `encoding/json/v2` is available experimentally (1.25+, `GOEXPERIMENT=jsonv2`) — mention it, don't default to it.
- Project layout consensus: `cmd/<binary>/main.go` for entry points, `internal/` for private packages (compiler-enforced), a flat root for small libraries. **Do not create `pkg/`** by reflex, don't mirror MVC layer names as packages (`models`, `utils`, `helpers` are smells); package names are part of the API (`storage.Client`, not `storage.StorageClient`).

## Decision frameworks with reasoning chains

**Channel or mutex?** Ask: (1) Is there a *transfer* — data moving from producers to consumers, stages of a pipeline, a completion signal? → channel. (2) Is it *state* — multiple goroutines reading/updating the same structure in place? → `sync.Mutex` (or `RWMutex` only with measured read-mostly contention; unmeasured RWMutex is premature). (3) Is it one-time signaling? → `close(ch)` broadcast, `sync.Once`, or context cancellation. (4) Is it a counter/flag? → `sync/atomic` types (`atomic.Int64`, `atomic.Bool`). What changes the answer: if the mutex-based design grows condition-waiting ("wait until the map has a key"), that's flow, switch to channels; if the channel design grows request/response pairs with reply channels everywhere, that's state access, switch to a mutex.
**Buffered or unbuffered?** Default unbuffered (synchronization point, backpressure by construction). Buffer size N only with a reason you can state: known burst size, decoupling producer hiccups, or a semaphore (`chan struct{}` of size N). "Buffer so sends don't block" is how you hide deadlocks and lose backpressure.

**Generics or interfaces?** (Generics have been in since 1.18; the judgment is mature now.) Use generics for: type-safe containers/collections, functions over `[]T` (`Map`, `Keys`), constraints like `cmp.Ordered` on algorithms. Use interfaces for: *behavioral* polymorphism — things that do something (`io.Reader`, `Storer`). Litmus test: if the type parameter appears only once in the signature and you never return it, an interface parameter is simpler (`func Log(w io.Writer)`, not `func Log[W io.Writer](w W)`). Avoid: generic structs to dodge writing two small concrete types, and `any`-constrained parameters that just re-create `interface{}` with extra syntax. The standard library's restraint (`slices`, `maps`, `cmp`) is the calibration.

**Performance investigation order.** Never guess: (1) `go test -bench . -benchmem` on the suspect path — allocations per op is the first number to read; (2) live services: `net/http/pprof` + `go tool pprof -http=: profile` — CPU profile for compute, heap profile (`inuse_space` for leaks, `alloc_space` for GC pressure), block/mutex profiles for contention, `goroutine` profile for leaks; (3) common wins in order of frequency: unpreallocated slices/maps in hot loops (`make([]T, 0, n)`), `[]byte`↔`string` conversions, fmt in hot paths, tiny interface-boxed allocations, JSON marshal cost. Stopping rule: stop when the top frame is your actual business logic or syscalls, or when p99 meets the SLO — shaving allocations below GC-noise level is waste.

## How an expert thinks through it: rising goroutine count

Dashboard shows goroutines climbing 50/hour, memory following. Internal monologue: *Classic leak. Get evidence before theory: hit `/debug/pprof/goroutine?debug=1` twice, 10 minutes apart, and diff the stacks. Prior: leaks cluster at (a) `chan send` — a worker writing results to a channel whose reader returned early (e.g., on error, or `select` took the ctx branch and abandoned the result channel); (b) `chan receive` on a never-closed channel; (c) blocked on network I/O with no deadline; (d) time.Tick — `time.Tick` leaks its ticker by design, must use `time.NewTicker` + `defer Stop()` in anything long-running (though since Go 1.23 unreferenced tickers are GC-able, a stopped-but-referenced loop still leaks the goroutine around it). Say the diff shows hundreds parked in `results <- r` inside `fetchOne`. So the spawner stopped receiving. Look at the collector: `for i := 0; i < n; i++ { select { case r := <-results: ...; case <-ctx.Done(): return } }` — there it is: on cancellation it returns, orphaning up-to-n senders on an unbuffered channel. Fixes considered: (1) make `results` buffered with capacity n — senders never block; leak fixed with one character. Accept: bounded, simple, idiomatic for known-cardinality fan-in. (2) Have senders `select` on ctx too — also correct, more code; do it if senders are long-lived. (3) errgroup rewrite — `golang.org/x/sync/errgroup` with `g.Go` + shared slice indexed per-worker removes the channel entirely; best if I'm touching this code anyway. Ship (1) now, (3) in the refactor. Verify: goroutine profile flat under load test with injected cancellations.* Prior to internalize: **fan-in + early return = orphaned senders; size result channels to the number of sends, or make senders cancellable.**

## Failure modes and pitfalls

- **Nil interface gotcha.** An interface is nil only if *both* its type and value are nil. `var p *MyErr; var err error = p; err != nil` is **true**. The bug ships as `func f() error { var e *MyErr; ...; return e }` — returns non-nil error wrapping a nil pointer. Rule: return the literal `nil`, never a possibly-nil concrete pointer through an interface return.
- **Slice aliasing and append sharing.** `b := a[:2]; b = append(b, x)` writes into `a`'s backing array if capacity allows — `a[2]` silently changes; if capacity is full, append reallocates and they diverge. Both behaviors are correct Go and both surprise. Defenses: full-slice expressions `a[low:high:max]` to cap capacity when handing out sub-slices; `slices.Clone` when the callee might append/mutate; never retain sub-slices of large buffers (pins the whole array — copy out what you keep).
- **Loop-variable capture is FIXED (Go 1.22+): each iteration gets a fresh variable** — the historical `go func(){ use(v) }()` bug no longer applies on supported versions. Don't "fix" it in new code; do still check `go.mod` says ≥1.22 before relying on it.
- **defer in loops.** `defer f.Close()` inside a range over 10k files runs all closes at function exit — fd exhaustion. Extract the body into a function or close explicitly.
- **Context misuse.** Ignoring ctx in a select (`case <-ch:` with no `case <-ctx.Done():`) makes cancellation advisory; forgetting `defer cancel()` from `context.WithTimeout` leaks the timer and its goroutine until expiry; using `context.WithValue` for parameters (a `userID` your function requires belongs in the signature).
- **Error-handling anti-patterns.** `if err != nil { log.Error(err); return err }` — double reporting; pick one. `errors.New` in hot comparison paths without a package-level sentinel (`var ErrNotFound = errors.New("not found")`) makes `errors.Is` impossible. Wrapping with `%v` instead of `%w` severs the chain. Exporting error *types* when a sentinel would do commits you to API surface.
- **WaitGroup misuse.** `wg.Add(1)` must happen *before* `go` (inside the goroutine is a race with `Wait`); passing WaitGroup by value copies it (vet catches this — run `go vet` always).
- **Map iteration order and races.** Order is deliberately randomized — any test depending on it flakes. Concurrent map writes are a fatal runtime crash, not a data corruption: guard with a mutex or use `sync.Map` only for the two blessed cases (append-mostly caches; disjoint key sets per goroutine).
- **Table-driven tests, the load-bearing conventions:** name each case (`tests := []struct{ name string; ... }`), use `t.Run(tt.name, ...)` for isolation and `-run 'TestX/case'` targeting, `t.Parallel()` where cases are independent, `got`/`want` vocabulary with `cmp.Diff` (google/go-cmp) for structs. Test through the public API of the package (`package foo_test`). For concurrency tests on 1.25+, use `testing/synctest` to make time deterministic instead of sleeps. Run `go test -race ./...` in CI unconditionally — the race detector is the single highest-value flag in the toolchain.
- **Interface pollution.** Defining interfaces "for mockability" on every type produces one-implementation interfaces everywhere. Mock at architectural boundaries (storage, external APIs) using small consumer-defined interfaces; test everything else with real types.

## Worked micro-examples

**Worker pool with a complete exit plan (the shape to memorize):**
```go
func process(ctx context.Context, jobs <-chan Job) error {
    g, ctx := errgroup.WithContext(ctx) // golang.org/x/sync/errgroup
    for range 8 {
        g.Go(func() error {
            for {
                select {
                case <-ctx.Done():
                    return ctx.Err()          // exit plan 1: cancellation
                case j, ok := <-jobs:
                    if !ok { return nil }      // exit plan 2: channel closed by producer
                    if err := handle(ctx, j); err != nil {
                        return fmt.Errorf("job %s: %w", j.ID, err) // cancels siblings via ctx
                    }
                }
            }
        })
    }
    return g.Wait() // exit plan 3: someone provably waits
}
```

**Consumer-side interface + error chain:**
```go
// package report (the CONSUMER defines what it needs)
type UserGetter interface{ GetUser(ctx context.Context, id string) (User, error) }

func Build(ctx context.Context, ug UserGetter, id string) (*Report, error) {
    u, err := ug.GetUser(ctx, id)
    if errors.Is(err, storage.ErrNotFound) { return nil, fmt.Errorf("report for %s: %w", id, ErrNoSubject) }
    if err != nil { return nil, fmt.Errorf("loading user %s: %w", id, err) }
    ...
}
// package storage returns *storage.Client (concrete); it never heard of UserGetter.
```

## Verification and stopping rule

Before presenting Go code or advice: (1) point to the exit plan of every goroutine (who stops it, who waits); (2) `go vet` and `-race` must be assumed — does the code survive both? (3) check every `append` on a shared slice and every returned sub-slice for aliasing; (4) confirm error chains use `%w` end-to-end and callers use `Is`/`As`, never `strings.Contains`; (5) version-gate advice: range-over-int and fixed loop vars need 1.22+, synctest needs 1.25+. Stop simplifying when the code reads top-to-bottom without goroutine bookkeeping in your head, interfaces have ≤3 methods, and pprof shows business logic on top — Go rewards stopping early; cleverness is the language's only real enemy.
