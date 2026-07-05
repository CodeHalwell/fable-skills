---
name: go-development
description: Loads expert Go judgment for goroutine lifecycle management, channels vs mutexes, context discipline, interface and error design, slice/nil gotchas, generics restraint, and pprof-driven performance work. Use when writing or reviewing Go services, debugging goroutine leaks or races, designing packages and interfaces, or structuring Go tests and project layout.
---

# Go Development

## Core mental model

1. **Every goroutine needs an exit plan before you write `go`.** Goroutine leaks are the #1 Go production bug class: a goroutine parked forever on a channel nobody reads, a context nobody cancels, a loop with no stop condition. Before every `go` statement, answer two questions — *how does this goroutine terminate?* and *who waits for it?* If either answer is missing, don't start it.
2. **Share memory by communicating — but only when there's a flow.** Channels model movement: pipelines, fan-out/fan-in, ownership hand-off, completion signals. Mutexes model state: caches, counters, config. The slogan is not "always channels" — a mutex around a map is simpler, faster, and more obvious than a channel-fronted map-owning goroutine.
3. **`context.Context` is the cancellation/deadline bus, and it flows one way.** First parameter of anything that blocks, does I/O, or can be slow: `func F(ctx context.Context, ...)`. Never stored in structs (except transitional request wrappers), never `nil` (use `context.Background()`), and its value bag is for request-scoped cross-cutting data (trace ID, auth principal) — not for function parameters in disguise.
4. **Interfaces are discovered by consumers, not designed by producers.** Define the interface in the package that *uses* it, sized to exactly the methods it calls — often one. Accept interfaces, return concrete types. A package exporting `FooInterface` next to `fooImpl` with a constructor returning the interface is Java smuggled into Go.
5. **Errors are values with a wrapping convention.** `fmt.Errorf("opening config: %w", err)` builds a cause chain; `errors.Is` (sentinels) and `errors.As` (types) walk it. Callers never string-match. Each error is *handled once*: log it or return it, not both.
6. **Clarity beats cleverness, structurally.** Go's design rewards code that reads top-to-bottom with no hidden control flow. When choosing between a clever abstraction and 15 lines of repetition, Go culture — and future debuggability — usually picks the repetition.

## Current state (verified July 2026)

- **Go 1.26 is current** (Feb 2026); 1.25 remains supported (Go supports the last two minors). Highlights that change advice:
  - **`testing/synctest` finalized in 1.25** — deterministic testing of concurrent code with a virtualized clock. Use it instead of `time.Sleep` in tests of timeout/retry logic.
  - **Container-aware `GOMAXPROCS` (1.25+)**: the runtime respects cgroup CPU limits — the `uber-go/automaxprocs` workaround is no longer needed on modern versions.
  - **Green Tea GC**: experimental in 1.25 (`GOEXPERIMENT=greenteagc`), **on by default in 1.26** — roughly 10–40% GC overhead reduction in GC-heavy programs; no code changes required.
  - `encoding/json/v2` exists behind `GOEXPERIMENT=jsonv2` (1.25+) — mention as future, don't default to it.
  - Loop variables are per-iteration since 1.22; `for range n` over ints also since 1.22.
- Layout consensus: `cmd/<binary>/` for entry points, `internal/` for private packages (compiler-enforced), flat root for small libraries. **Do not create `pkg/` by reflex.** Package names are API: `storage.Client` not `storage.StorageClient`; no `utils`, `helpers`, `common`, `models` grab-bags — packages are organized by *capability*, not by layer.

## Decision frameworks with reasoning chains

**Channel or mutex?** Ask in order:
1. Is there a *transfer* — data moving producer→consumer, pipeline stages, a completion signal? → channel.
2. Is it *state* — goroutines reading/updating a structure in place? → `sync.Mutex`. (`RWMutex` only with *measured* read-mostly contention; unmeasured RWMutex is premature and can be slower.)
3. One-time broadcast? → `close(ch)`, `sync.Once`, or context cancellation.
4. Counter or flag? → `sync/atomic` types (`atomic.Int64`, `atomic.Bool`).
What flips the answer: a mutex design growing "wait until condition" logic is flow — switch to channels; a channel design growing request/response pairs with reply channels everywhere is state access — switch to a mutex. Both migrations are common and both directions are legitimate.

**Buffered or unbuffered?** Default unbuffered: it's a synchronization point and gives backpressure by construction. Choose buffer size N only with a stateable reason: known burst size, decoupling a jittery producer, a semaphore (`make(chan struct{}, N)`), or fan-in sized to the number of senders (see scenario). "Buffered so sends don't block" is how deadlocks get hidden and backpressure gets lost.

**Generics or interfaces?** (Generics landed in 1.18; judgment is mature.)
- Generics: type-safe containers, functions over slices/maps (`Map`, `Keys`, `Dedup`), algorithms with `cmp.Ordered` constraints. The `slices`, `maps`, `cmp` stdlib packages are the calibration for "worth it."
- Interfaces: *behavioral* polymorphism — things that do (`io.Reader`, `Storer`).
- Litmus test: if the type parameter appears once in the signature and is never returned, use an interface parameter instead — `func Log(w io.Writer)`, not `func Log[W io.Writer](w W)`.
- Refuse: generic structs to avoid writing two small concrete types; `any`-constrained parameters recreating `interface{}` with ceremony; premature constraint hierarchies. If the pre-1.18 answer would have been "just write it twice," that often remains the answer.

**Performance investigation order.** Never guess:
1. `go test -bench . -benchmem` on the suspect path — read allocs/op before ns/op.
2. Live services: import `net/http/pprof`, then `go tool pprof -http=: <url>/debug/pprof/profile` (CPU), heap profile — `inuse_space` for leaks, `alloc_space` for GC pressure — block/mutex profiles for contention, goroutine profile for leaks.
3. Wins in order of base rate: unpreallocated slices/maps in hot loops (`make([]T, 0, n)`), `[]byte`↔`string` churn, `fmt` in hot paths, interface-boxing allocations, JSON marshal cost.
Stopping rule: stop when the top frames are business logic or syscalls, or p99 meets SLO. Shaving allocations below GC noise is waste — and on 1.26 the GC is cheaper, so re-measure before porting old folklore.

## How an expert thinks through it: rising goroutine count

Dashboard: goroutines +50/hour, memory tracking it.

Internal monologue: *Classic leak; get stacks before theories. Hit `/debug/pprof/goroutine?debug=1` twice, 10 minutes apart, diff the counts per stack. Priors, by base rate: (a) parked on `chan send` — worker writing results to a channel whose reader bailed early; (b) parked on `chan receive` from a never-closed channel; (c) network I/O with no deadline set (http.Client without Timeout, raw conns without SetDeadline); (d) tickers — `time.Tick` has no Stop and historically leaked by design; use `time.NewTicker` + `defer Stop()` (since 1.23 unreferenced tickers can be GC'd, but the goroutine looping around one still leaks if it has no exit). The diff shows 400 goroutines in `results <- r` inside `fetchOne` — so senders outlived their reader. Find the collector: `for i := 0; i < n; i++ { select { case r := <-results: ...; case <-ctx.Done(): return } }` — there: on cancellation it returns, orphaning up to n senders on an unbuffered channel, forever. Fixes considered: (1) `make(chan result, n)` — buffer to the number of sends; senders can always complete; one-line fix, idiomatic for known-cardinality fan-in. (2) Senders also select on ctx — correct, more code; right when senders are long-lived or n is unbounded. (3) Rewrite with `errgroup.WithContext` — removes the manual channel entirely; best if I'm refactoring anyway. Rejected: increasing the pod memory limit (treats the symptom); a watchdog that kills "old" goroutines (no such primitive, and the desire for one signals design failure). Ship (1) today, schedule (3).*

Verification: load test with injected cancellations; goroutine profile must return to baseline. Prior to internalize: **fan-in + early return = orphaned senders. Size result channels to the number of sends, or make every send cancellable.**

## Second scenario: intermittent test failure, "concurrent map writes"

CI crashes once in ~30 runs: `fatal error: concurrent map writes` in a cache package that "has a mutex."

Internal monologue: *This is a fatal runtime check, not a flake — the race is real and the mutex has a gap. Don't eyeball; make the detector find it: `go test -race -count=50 ./cache/...` locally. The race detector reports the two racing goroutines with stacks — say, `Get` writing under `RLock`. There it is: `func (c *Cache) Get(k string) V { c.mu.RLock(); defer c.mu.RUnlock(); if v, ok := c.m[k]; ok { return v }; v := c.load(k); c.m[k] = v; return v }` — a write under a* read *lock. Fix options: (1) take the write lock for the whole Get — correct, simplest, serializes all reads; fine unless profiling shows contention. (2) Double-checked pattern: RLock read; miss → RUnlock, Lock, re-check (another goroutine may have filled it), fill, Unlock — standard, more code, still calls `load` under the write lock (bad if load is slow I/O). (3) `singleflight.Group` (golang.org/x/sync) — dedups concurrent loads per key without holding the map lock during I/O; the right answer when `load` is expensive. (4) `sync.Map` — rejected: this is read-modify-write with loads, not one of its blessed workloads. Choose (1) if load is cheap, (3) if load does I/O. Also fix process: `-race` was not in CI, which is how this shipped — that's the root cause; the code bug is a symptom.*

Prior: **"has a mutex" is not "is correct" — audit that every write path holds the* write *lock, and put `-race` where it runs on every merge, not on the developer's laptop.**

## Failure modes and pitfalls

- **Nil interface gotcha.** An interface is nil only when *both* its type and value are nil. `var p *MyErr; var err error = p; err != nil` is **true** — a typed nil inside a non-nil interface. Ships as: `func f() error { var e *MyErr; ...; return e }` — callers see a non-nil error that explodes on use. Rule: return literal `nil`, never a possibly-nil concrete pointer through an interface return type.
- **Slice aliasing and append sharing.** `b := a[:2]; b = append(b, x)` writes into `a`'s backing array when capacity allows — `a[2]` silently changes; when capacity is full, append reallocates and the slices diverge. Both are correct Go; both surprise. Defenses: full-slice expressions `a[low:high:max]` to cap capacity on handed-out sub-slices; `slices.Clone` before a callee might append; never retain a small sub-slice of a huge buffer (pins the whole array — copy out).
- **Loop variables are per-iteration since Go 1.22** — the historical `go func(){ use(v) }()` capture bug is dead on supported versions. Don't add `v := v` to new code; do check `go.mod` says ≥1.22 before deleting it from old code.
- **`defer` in loops.** `defer f.Close()` inside a 10k-file range runs every close at *function* exit — fd exhaustion. Extract the loop body into a function, or close explicitly.
- **Context misuse.** A `select` without a `case <-ctx.Done():` makes cancellation advisory; forgetting `defer cancel()` after `context.WithTimeout` leaks the timer goroutine until expiry; `context.WithValue` for required parameters (a `userID` your function needs belongs in its signature — values are for cross-cutting metadata).
- **Error anti-patterns.** `log.Error(err); return err` — double handling, pick one. Wrapping with `%v` instead of `%w` — severs the chain, `errors.Is` goes blind. Ad-hoc `errors.New` at each call site instead of a package sentinel (`var ErrNotFound = errors.New("not found")`) — makes matching impossible. Exporting error *types* when a sentinel suffices — needless API surface.
- **WaitGroup rules.** `wg.Add(1)` *before* `go`, never inside the goroutine (races with `Wait`); pass `*sync.WaitGroup` or capture it — copying is a vet error. Run `go vet ./...` always; it's free and catches this class.
- **Map behavior.** Iteration order is deliberately randomized — tests depending on it flake; sort keys first. Concurrent map writes are a *fatal runtime crash*, not silent corruption: guard with a mutex; `sync.Map` only for its two blessed workloads (write-once/read-many caches; disjoint key sets per goroutine).
- **Goroutines in HTTP handlers.** `go doSlowThing(r.Context())` — the request context is cancelled when the handler returns, killing the background work; and nothing waits for the goroutine at shutdown. Background work needs a detached context (`context.WithoutCancel` since 1.21) *and* registration with the server's shutdown WaitGroup/errgroup.
- **Interface pollution.** An interface per struct "for mockability" produces one-implementation interfaces everywhere and hides the concrete type's docs. Mock at architectural boundaries (storage, external APIs) via small consumer-defined interfaces; test everything else with real types.
- **Table-driven tests, the load-bearing conventions:** named cases in `[]struct{ name string; ... }`; `t.Run(tt.name, ...)` for isolation and `-run 'TestX/case'` targeting; `t.Parallel()` where independent; `got`/`want` with `cmp.Diff` (google/go-cmp) for structs; test the *public* API from `package foo_test`; `t.Helper()` in assertion helpers so failures point at the case. Concurrency/time logic on 1.25+: `testing/synctest` with its fake clock instead of real sleeps. CI: `go test -race ./...` unconditionally — the race detector is the highest-value flag in the toolchain.

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
                    return ctx.Err()               // exit plan 1: cancellation
                case j, ok := <-jobs:
                    if !ok {
                        return nil                 // exit plan 2: producer closed the channel
                    }
                    if err := handle(ctx, j); err != nil {
                        return fmt.Errorf("job %s: %w", j.ID, err) // fails group, cancels siblings
                    }
                }
            }
        })
    }
    return g.Wait()                                // exit plan 3: someone provably waits
}
```

**Consumer-side interface + error chain across layers:**
```go
// package report — the CONSUMER defines the interface it needs, sized to use
type UserGetter interface {
    GetUser(ctx context.Context, id string) (User, error)
}

func Build(ctx context.Context, ug UserGetter, id string) (*Report, error) {
    u, err := ug.GetUser(ctx, id)
    switch {
    case errors.Is(err, storage.ErrNotFound):
        return nil, fmt.Errorf("report for %s: %w", id, ErrNoSubject) // translate at the boundary
    case err != nil:
        return nil, fmt.Errorf("loading user %s: %w", id, err)       // wrap with context, %w
    }
    _ = u
    // ...
    return &Report{}, nil
}
// package storage returns *storage.Client (concrete). It has never heard of UserGetter —
// *storage.Client satisfies it structurally. That's the whole design.
```

**Table-driven test skeleton:**
```go
func TestParse(t *testing.T) {
    tests := []struct {
        name    string
        in      string
        want    Config
        wantErr error
    }{
        {name: "defaults", in: "{}", want: DefaultConfig()},
        {name: "bad port", in: `{"port":-1}`, wantErr: ErrBadPort},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got, err := Parse(strings.NewReader(tt.in))
            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("Parse() error = %v, want %v", err, tt.wantErr)
            }
            if diff := cmp.Diff(tt.want, got); diff != "" {
                t.Errorf("Parse() mismatch (-want +got):\n%s", diff)
            }
        })
    }
}
```

**HTTP server with the timeouts that don't default (and a shutdown path):**
```go
func main() {
    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second,   // zero value = slowloris-vulnerable
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second, // keep-alive reaping
    }
    go func() {
        if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
            log.Fatalf("serve: %v", err)
        }
    }()

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, os.Interrupt)
    defer stop()
    <-ctx.Done()                              // block until shutdown signal

    shutCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutCtx); err != nil { // drains in-flight requests
        log.Printf("forced shutdown: %v", err)
    }
}
// Same rule client-side: never use http.DefaultClient in production —
// it has NO timeout; construct &http.Client{Timeout: 10 * time.Second} or use
// per-request contexts with http.NewRequestWithContext.
```

## Verification and stopping rule

Before presenting Go code or advice:
1. Point to the exit plan of every goroutine — who stops it, who waits for it. No answer, no `go`.
2. Assume `go vet ./...` and `go test -race ./...` run — does the code survive both?
3. Check every `append` on shared or returned slices for aliasing, and every interface-typed return for the typed-nil trap.
4. Confirm error chains use `%w` end-to-end, sentinels exist where callers match, and nothing string-matches error text.
5. Version-gate: per-iteration loop vars and range-over-int need 1.22+; `synctest` needs 1.25+; Green-Tea-GC-by-default claims need 1.26.
6. For any performance assertion, name the profile or benchmark that would confirm it — if you can't, present it as a hypothesis to measure, not a fact.

Stopping rule: done when the code reads top-to-bottom without goroutine bookkeeping in your head, interfaces have ≤3 methods and live in consumer packages, and pprof puts business logic on top. Go punishes cleverness more than any mainstream language — when in doubt, write the dumber version and stop.
