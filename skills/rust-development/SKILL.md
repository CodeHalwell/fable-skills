---
name: rust-development
description: Loads expert Rust judgment for ownership-driven design, lifetimes, smart-pointer and error-handling architecture, async pitfalls, trait design, and unsafe discipline. Use when writing or reviewing Rust, resolving borrow-checker fights, choosing Box/Rc/Arc/RefCell/Mutex, structuring errors with thiserror/anyhow, or debugging Send/lifetime errors in async code.
---

# Rust Development

## Core mental model

1. **The borrow checker is a design review, not an obstacle.** When it rejects your code, it has usually found one of: shared mutable state you didn't admit to, an unclear owner, or a lifetime you couldn't explain to a colleague either. The expert response to a borrow error is to *restructure ownership* (split structs, pass indices, take by value, narrow borrow scopes), not to reach for `clone()`, `Rc<RefCell<...>>`, or `unsafe`. Fighting the checker with escape hatches moves compile errors to runtime panics.
2. **Ownership design precedes coding.** Decide before writing: who owns each piece of data, who borrows it and for how long, and what is shared across threads. Rust programs are trees of ownership with borrows as temporary edges; graphs (parents pointing at children pointing back) require deliberate machinery (indices/arenas/`Weak`), so avoid modeling them casually.
3. **Every escape hatch trades a static guarantee for a dynamic one.** `RefCell` converts compile-time borrow errors into runtime panics; `Mutex` converts them into potential deadlocks; `unsafe` converts them into UB. Each step down needs written justification.
4. **Types encode state machines.** Prefer making invalid states unrepresentable (enums with data, newtypes, typestate builders) over runtime checks. `Option<T>` in a struct field is a question: is "absent" a real domain state or a construction-order artifact? If the latter, restructure (builder or two types) instead.
5. **Async Rust is cooperative state machines, not green threads.** A future does nothing until polled; dropping it *is* cancellation, and it can happen at any `await` point. This one fact explains most async production bugs.

## Current state (verified July 2026)

- Stable Rust is in the **1.96.x** line (1.96.1, June 2026). **Edition 2024** is the production default (stabilized in 1.85): async closures (`async || {}`), `impl Trait` lifetime-capture changes (`use<...>` syntax), `unsafe_op_in_unsafe_fn` by default, cargo resolver v3.
- Async ecosystem consolidated: **tokio is the default runtime** (LTS lines maintained, e.g. 1.47/1.51); **async-std is discontinued** — flag it if you see it in a dependency tree; smol survives for lightweight/embedded-ish cases. Web: axum is the default server framework choice; error crates: thiserror + anyhow remain standard (eyre for fancier reports).

## Decision frameworks with reasoning chains

**Smart-pointer chain — ask these in order, take the first yes:**
1. *Do I even need a pointer?* Plain ownership `T` or a borrow `&T`/`&mut T` covers most cases. Borrow errors → first try restructuring (below), not pointers.
2. *Is it just too big / recursive / dyn?* → `Box<T>`. Sole ownership, heap, no sharing.
3. *Multiple owners, single thread?* → `Rc<T>`. Immutable shared.
4. *Multiple owners across threads?* → `Arc<T>`.
5. *Shared AND mutable?* Now compose interior mutability inside 3/4: single-threaded → `Rc<RefCell<T>>` (runtime borrow panics possible); cross-thread → `Arc<Mutex<T>>` or `Arc<RwLock<T>>` (RwLock only when reads dominate massively and you've considered writer starvation); a single scalar → `Arc<AtomicUsize>` etc.
6. *Cycles?* Break one direction with `Weak`. Parent↔child = parent owns `Vec<Rc<Child>>`, child holds `Weak<Parent>`.
What updates the decision: reaching step 5 in *application logic* (not a cache/registry) is a smell — usually the real fix is message passing (channels) or restructuring so one owner mutates. Priors: `Box` is common, `Arc` is common at architecture seams, `Rc<RefCell>` should be rare and mostly appears in graph/GUI/interpreter code.

**Lifetime annotation decision process.** Don't start by writing lifetimes; start by asking *should this type borrow at all?* (1) Elision covers most functions — write none and see. (2) If the compiler asks, the question it's really asking is "which input does the output borrow from?" — answer that honestly (`fn pick<'a>(a: &'a str, b: &str) -> &'a str`). (3) A struct holding `&'a T` infects every user with `'a`; only do this for short-lived views (parsers, iterators, guards). For anything stored, cached, or sent across await points/threads, own the data (`String`, not `&str`) or `Arc` it. (4) `'static` bound on generics means "no non-static borrows inside," not "lives forever" — owned `String` satisfies `T: 'static`. When a lifetime puzzle exceeds ~10 minutes, that's the signal the design wants owning types.

**Error architecture: thiserror vs anyhow boundary.** The rule: **libraries and recoverable layers define typed errors (thiserror); binaries and top-level orchestration use anyhow.** Reasoning: a caller can only match on variants you export — if any caller needs to distinguish `NotFound` from `Conflict`, that layer needs an enum. If the only consumer is a log line or a process exit code, `anyhow::Result` with `.context("reading config")` is strictly better (less code, richer messages). Conversion happens once, at the boundary, via `?` (anyhow absorbs any `std::error::Error`). Anti-patterns: anyhow in a published library's public API (forces downstream to string-match); a giant crate-wide error enum with 30 variants nobody matches (that's anyhow with extra steps — split per-module or collapse).

**Trait objects vs generics.** Generics (`impl Trait`/`<T: Trait>`) = static dispatch, monomorphized, zero-cost, but code bloat and infectious signatures. `dyn Trait` = one compiled copy, heterogeneous collections (`Vec<Box<dyn Handler>>`), runtime-chosen implementations, at the cost of vtable indirection and dyn-compatibility limits (no generic methods, no `Self`-returning methods unless boxed). Default to generics for hot paths and simple bounds; switch to `dyn` when you need heterogeneity, plugin registration, or compile-time relief. Coherence/orphan rule: you can implement a trait only if you own the trait or the type — plan for this by exporting traits from lower crates in a workspace, or use the newtype wrapper pattern to implement foreign traits on foreign types.

## How an expert thinks through it: a borrow-checker fight

`self.apply(&self.rules[i])` fails: cannot borrow `*self` as mutable because `self.rules` is also borrowed. Internal monologue: *The checker sees `&mut self` for `apply` overlapping an immutable borrow into `self.rules`. It's right that the signatures conflict, even if `apply` never touches `rules`. Options: (a) clone the rule — works if `Rule` is small/cheap; acceptable, slightly smelly if in a hot loop. (b) Split the struct: move `rules` into its own field-group so I can borrow disjoint fields — the checker splits borrows per-field, so `Self { rules, engine }` with `engine.apply(&rules[i])` compiles with zero cost. This is the canonical fix: the compiler was telling me `apply` shouldn't take all of `self`. (c) Indices: pass `i` and let `apply` fetch — just moves the conflict inside. (d) `Rc<RefCell<Rule>>` — absolutely not; converts a design smell into a runtime panic class. Choose (b); it also improves the API because `apply`'s real dependencies are now explicit.* The generalized prior: **"cannot borrow X because Y" across method calls is usually a method taking `&mut self` when it needs only some fields — split the struct or take the needed fields as parameters.**

## Failure modes and pitfalls

- **Clone-to-appease.** Sprinkling `.clone()` until it compiles. Each clone is fine alone; the pattern means ownership was never designed. In review, ask of every clone: who is the real owner? Often a `&` reshuffle or moving a `let` earlier removes it.
- **GC-refugee errors** (from Java/Go/Python/JS): modeling object soups with `Rc<RefCell<Node>>` graphs (use `Vec` + indices or `petgraph`/`slotmap` arenas); expecting mutation through shared references (Rust requires exclusive `&mut` — aliasing XOR mutation); storing borrowed strings in long-lived structs; implementing OO inheritance with trait objects + default methods instead of composition/enums; overusing `String` where `&str` parameters suffice (`fn f(s: &str)`, callers pass `&owned`).
- **Holding a std `Mutex` guard across `.await`.** `std::sync::MutexGuard` is not `Send`, so the compiler usually stops you (with a confusing "future cannot be sent between threads" error pointing at `tokio::spawn`) — the *fix* is to end the lock scope before awaiting (`let v = { lock.x }; do_async(v).await`), or use `tokio::sync::Mutex` only when the guard genuinely must live across an await. Prefer std Mutex + short critical sections; it's faster and deadlock-safer.
- **Reading `Send` errors backwards.** "future is not Send" at a `spawn` call is almost never about the spawned function itself — scan the async body for what lives across an `await`: `Rc`, `RefCell` guards, `MutexGuard`, raw pointers. Drop or scope them before the await point.
- **Cancellation-unsafety.** Any future can be dropped at an await point — in `tokio::select!`, the losing branches are dropped every iteration. Bugs: half-completed multi-step writes; `recv()` futures on some channel types losing a claimed message when cancelled; state mutated before an await and never rolled back. Defenses: make each await-separated step idempotent or transactional; in select loops, prefer cancel-safe operations (documented per-method in tokio docs); do multi-step work in a spawned task that owns its data rather than in a select arm.
- **Blocking the runtime.** CPU-heavy work or sync I/O in an async fn stalls a worker thread; enough of them stalls the runtime. Use `tokio::task::spawn_blocking` for sync/CPU work (or a rayon pool for data-parallel compute, bridged with a oneshot channel). Symptom to recognize: timers and unrelated requests get slow together.
- **`async fn` in public traits and dyn.** Native `async fn` in traits works for generic-dispatch cases, but such traits aren't directly dyn-compatible; for `dyn` use, use the `async-trait` crate (boxing) or hand-written `-> Pin<Box<dyn Future>>` methods. Also, in public APIs, `async fn` in traits leaks auto-trait uncertainty — libraries often want explicit `impl Future + Send` returns.
- **Unsafe discipline.** Every `unsafe` block gets a `// SAFETY:` comment stating the invariant and why it holds *at this call site*. Keep unsafe minimal and wrap it in a safe API whose types enforce the invariant; the module boundary is the soundness boundary — audit everything in the module that can touch the invariant, not just the block. Run **Miri** (`cargo +nightly miri test`) on any crate with nontrivial unsafe; it catches UB (aliasing violations, use-after-free) tests can't. Edition 2024's `unsafe_op_in_unsafe_fn` means unsafe fns need internal unsafe blocks — don't silence it.
- **Feature-flag misuse in workspaces.** Cargo features must be *additive* (union of features across the dependency graph gets compiled — features are unified per crate); a `no-std` or "exclusive backend" modeled as default-on features breaks when two dependents disagree. Pattern: `default = []`-ish minimal defaults in libraries, additive opt-ins, `dep:` syntax for optional dependencies, and remember `--no-default-features` in CI to catch accidental reliance. In workspaces, put shared versions in `[workspace.dependencies]` and inherit with `package.workspace = true`; note `cargo test --workspace` uses feature unification that can mask per-crate feature bugs — test leaf crates in isolation occasionally.

## Worked micro-examples

**The error-architecture boundary:**
```rust
// library crate: typed, matchable
#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("key not found: {0}")]
    NotFound(String),
    #[error("backend io")]
    Io(#[from] std::io::Error),
}
pub fn get(k: &str) -> Result<Vec<u8>, StoreError> { /* ... */ # todo!() }

// binary crate: context-rich, untyped
use anyhow::Context;
fn main() -> anyhow::Result<()> {
    let cfg = std::fs::read_to_string("app.toml").context("reading app.toml")?;
    let v = store::get("user:1").context("loading user")?; // StoreError -> anyhow via ?
    Ok(())
}
```

**Cancel-safe select loop shape:**
```rust
loop {
    tokio::select! {
        biased;                       // check shutdown first, deterministically
        _ = shutdown.changed() => break,
        Some(job) = rx.recv() => {    // tokio mpsc recv is cancel-safe
            // do NOT await multi-step work here; a shutdown would drop it midway
            tracker.spawn(handle(job));  // task owns the job to completion
        }
    }
}
tracker.close(); tracker.wait().await;   // tokio_util::task::TaskTracker
```

## Verification and stopping rule

Before presenting Rust code or advice: (1) `cargo clippy --all-targets -- -D warnings` mentally — clippy catches most idiom violations you'd otherwise ship; (2) for every `clone`, `Rc<RefCell<_>>`, and `unwrap`, confirm a one-line justification exists; (3) for async code, walk each `await` and ask "what happens if the future is dropped right here?" and "what is alive across this point that isn't Send?"; (4) for unsafe, the SAFETY comment must cite a checkable invariant, and Miri should run in CI; (5) don't claim performance characteristics without `cargo bench`/criterion numbers. Stop refactoring when ownership reads as a tree, public APIs take borrows and return owned (or entirely owned at async/thread seams), and errors are typed exactly where a caller matches on them — chasing zero-clone purity in cold paths is waste.
