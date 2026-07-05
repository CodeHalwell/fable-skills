---
name: rust-development
description: Loads expert Rust judgment for ownership-driven design, lifetimes, smart-pointer and error-handling architecture, async pitfalls, trait design, and unsafe discipline. Use when writing or reviewing Rust, resolving borrow-checker fights, choosing Box/Rc/Arc/RefCell/Mutex, structuring errors with thiserror/anyhow, or debugging Send/lifetime errors in async code.
---

# Rust Development

## Core mental model

1. **The borrow checker is a design review, not an obstacle.** When it rejects code, it has usually found one of: shared mutable state you didn't admit to, an unclear owner, or a lifetime you couldn't explain to a colleague either. The expert response to a borrow error is to *restructure ownership* — split structs, narrow borrow scopes, pass indices, take by value — not to reach for `clone()`, `Rc<RefCell<_>>`, or `unsafe`. Fighting the checker with escape hatches converts compile errors into runtime panics.
2. **Ownership design precedes coding.** Before writing, decide: who owns each piece of data, who borrows it and for how long, what crosses threads. Rust programs are ownership *trees* with borrows as temporary edges. Graphs — parents and children pointing at each other — require deliberate machinery (indices, arenas, `Weak`); never model them casually.
3. **Every escape hatch trades a static guarantee for a dynamic one.** `RefCell` converts compile-time borrow errors into runtime panics; `Mutex` converts them into potential deadlocks; `unsafe` converts them into UB. Each step down the ladder needs a written justification.
4. **Types encode state machines.** Make invalid states unrepresentable: enums with data instead of flag+optional-fields structs, newtypes for units and IDs, typestate builders for construction-order constraints. An `Option<T>` field is a question — is "absent" a real domain state, or a construction-order artifact? If the latter, restructure (builder, or two types) instead of unwrapping everywhere.
5. **Async Rust is cooperative state machines, not green threads.** A future does nothing until polled, and *dropping it is cancellation* — which can happen at any `await` point. This single fact explains the majority of async production bugs (half-done work, lost messages, broken invariants).
6. **Zero-cost abstraction has a compile-time and cognitive price.** Generics monomorphize (fast, bloaty, infectious signatures); trait objects indirect (flexible, one copy). Neither is free; choose per call-site-count and heterogeneity, not ideology.

## Current state (verified July 2026)

- Stable Rust is the **1.96.x** line (1.96.1, June 2026). **Edition 2024** is the production default (stabilized in 1.85): async closures (`async || {}`), revised `impl Trait` lifetime capture with `use<...>` syntax, `unsafe_op_in_unsafe_fn` by default, cargo resolver v3.
- Async ecosystem, consolidated: **tokio is the default runtime** (LTS lines exist, e.g. 1.47/1.51 with multi-year support windows); **async-std is discontinued** — treat it as a migration flag in any dependency tree; smol survives in the lightweight niche. Web servers default to **axum**; error handling defaults to **thiserror + anyhow** (eyre if you want fancier reports); serialization to serde (serde_json), CLI to clap.
- Runtime choice framework: pick tokio unless you have a stated reason (embedded/no-std constraints, tiny binaries → smol/embassy). Do not build libraries hard-wired to a runtime when feasible — accept `impl Future`, use runtime-agnostic primitives, and feature-gate tokio-specific integration.

## Decision frameworks with reasoning chains

**Smart-pointer selection — ask in order, take the first yes:**
1. *Do I need a pointer at all?* Plain `T`, `&T`, `&mut T` cover most code. A borrow error here → try restructuring first (see the scenario below), not pointers.
2. *Just too big, recursive, or `dyn`?* → `Box<T>`. Sole ownership, heap placement, nothing shared.
3. *Multiple owners, one thread?* → `Rc<T>` (immutable sharing).
4. *Multiple owners, across threads?* → `Arc<T>`.
5. *Shared AND mutable?* Compose interior mutability inside 3/4:
   - single-thread → `Rc<RefCell<T>>` — accept that aliased `borrow_mut()` now panics at runtime;
   - cross-thread → `Arc<Mutex<T>>`; `Arc<RwLock<T>>` only when reads dominate *measurably* and you've considered writer starvation;
   - one scalar → `Arc<AtomicUsize>` / `AtomicBool` etc.
6. *Cycles?* Break one direction with `Weak` — parent owns `Vec<Rc<Child>>`, child holds `Weak<Parent>`.

Priors and update rules: `Box` is routine; `Arc` is routine at architecture seams (config, shared clients); reaching step 5 in *application logic* (not a cache/registry/GUI) is a smell whose usual fix is message passing (channels) or giving one owner the mutation rights. `Rc<RefCell<_>>` appearing more than a couple times in a design means you're writing Java in Rust — stop and re-draw the ownership tree.

**Lifetime annotation decision process.** Don't start by writing lifetimes; start by asking whether the type should borrow at all.
1. Write no annotations; elision covers most functions.
2. When the compiler asks, it is asking exactly one question: *which input does the output borrow from?* Answer it honestly: `fn pick<'a>(a: &'a str, b: &str) -> &'a str`.
3. A struct holding `&'a T` infects every user with `'a`. Reserve borrowing structs for short-lived views: parsers, iterators, guards. Anything stored, cached, sent across threads or `await` points → own the data (`String` over `&str`) or `Arc` it.
4. `T: 'static` means "contains no non-static borrows," not "lives forever" — owned `String` satisfies it. Most `'static`-bound errors from `tokio::spawn` are fixed by moving/cloning owned data in, not by lifetime surgery.
5. Escalation stop: if a lifetime puzzle exceeds ~10 minutes, that is the design telling you to switch to owned types. Cloning a `String` at a boundary costs nanoseconds; a `'a` in a public API costs every downstream user forever.

**Error architecture — the thiserror/anyhow boundary.** Rule: **libraries and layers with matching callers define typed errors (thiserror); binaries and orchestration use anyhow.**
- Reasoning chain: a caller can only react to variants you export. Ask "will any caller branch on this failure?" If yes (`NotFound` vs `Conflict` → 404 vs 409), that layer needs an enum with `#[derive(thiserror::Error)]`. If the only consumer is a log line or exit code, `anyhow::Result` + `.context("reading config")` is strictly better — less code, richer messages.
- Conversion happens once, at the boundary, via `?` (anyhow absorbs any `std::error::Error`).
- Anti-patterns: anyhow in a published library's public API (downstream must string-match); one crate-wide 30-variant enum nobody matches on (that's anyhow with extra steps — split per module or collapse to anyhow); `Box<dyn Error>` in new code (anyhow does the same job with context and backtraces).

**Trait objects vs generics.**
- Generics (`<T: Trait>` / `impl Trait`): static dispatch, inlining, zero runtime cost; price is compile time, binary bloat, and infectious signatures.
- `dyn Trait`: one compiled copy, heterogeneous collections (`Vec<Box<dyn Handler>>`), runtime registration; price is vtable indirection and dyn-compatibility limits (no generic methods; `Self`-returning methods need boxing).
- Default: generics for hot paths and simple bounds; `dyn` for plugin registries, heterogeneity, and compile-time relief in big crates.
- Coherence/orphan rule: you may implement a trait only if you own the trait or the type. Plan around it: define traits in low-level crates of a workspace so higher crates can implement them; wrap foreign types in newtypes to implement foreign traits (`struct Meters(f64); impl Display for Meters`).

## How an expert thinks through it: a borrow-checker fight

`self.apply(&self.rules[i])` fails: *cannot borrow `*self` as mutable because `self.rules` is also borrowed as immutable*.

Internal monologue: *The checker sees `apply(&mut self, ...)` overlapping an immutable borrow into `self.rules`. It's right about the signatures even if `apply` never touches `rules` — signatures are the contract, bodies don't matter. Options: (a) clone the rule — compiles; fine if `Rule` is small; smelly in a hot loop and it papers over the real finding. (b) Split the struct: the checker splits borrows* per field*, so if `apply` lives on an `Engine` field and rules sit beside it — `self.engine.apply(&self.rules[i])` — disjoint field borrows compile with zero runtime cost. This is the canonical fix: the compiler was telling me `apply` should never have taken all of `self`. (c) Pass the index and re-fetch inside `apply` — just moves the conflict inside the method. (d) `Rc<RefCell<Rule>>` — absolutely not: converts a design smell into a latent panic class. Choose (b); as a bonus the API now states `apply`'s true dependencies.*

Generalized prior: **"cannot borrow X because Y is borrowed" across a method call usually means a method takes `&mut self` while needing only some fields — split the struct, or make the method a free function taking exactly the fields it uses.** Second most common: a long-lived `let` binding holding a borrow past its last use — shrink its scope with a block.

## Second scenario: "future cannot be sent between threads safely"

`tokio::spawn(handle_conn(state.clone(), sock))` errors with a page of trait bounds ending in `required by a bound in tokio::spawn`.

Internal monologue: *Don't read the headline; find the compiler note that says "captured value is not Send" or "future is not Send as this value is used across an await" — rustc names the exact value and await point. It points at `guard`, a `std::sync::MutexGuard`, alive across `stream.write_all(...).await`. Three candidate fixes, cheapest first: (1) Scope the guard: compute what I need, drop the lock, then await — `let payload = { let g = state.lock().unwrap(); g.render() }; stream.write_all(&payload).await?;`. Almost always right: critical sections should be short anyway, and awaiting while holding a lock is a latency/deadlock hazard independent of Send. (2) `tokio::sync::Mutex` — its guard is Send, so it "fixes" the error, but now every lock is an await point and I've legitimized holding locks across I/O; only correct when the protected operation is itself async and must be exclusive (e.g., a serialized connection). (3) `spawn_local`/`LocalSet` to avoid Send entirely — rejected: this is a multi-threaded server; pinning work to one thread to dodge a lock-scoping bug is backwards. Take (1). While here, check the other classic culprits I'd look for if the note had been vaguer: `Rc` (use `Arc`), `RefCell` (use `Mutex`/redesign), a `!Send` client library handle held across awaits (create it per-task instead).*

Prior: **Send errors are lifetime-of-a-value questions in disguise — the fix is almost always "make the non-Send thing die before the await," not "find a Send version of it."**

## Failure modes and pitfalls

- **Clone-to-appease.** Sprinkling `.clone()` until it compiles. Each clone is individually fine; the pattern means ownership was never designed. Review question for every clone: "who is the real owner here?" Often a `&` reshuffle or an earlier `let` removes it. (But: a deliberate clone at an API boundary to keep lifetimes out of a public type is good engineering — clones are a smell, not a sin.)
- **GC-refugee errors** (arriving from Java/Go/Python/JS):
  - Modeling object soups as `Rc<RefCell<Node>>` graphs → use `Vec` + indices, or `slotmap`/`petgraph` arenas.
  - Expecting mutation through shared references → Rust is aliasing XOR mutation; design for one mutator.
  - Storing `&str` in long-lived structs → own `String` at storage boundaries.
  - Simulating inheritance with trait objects + default methods → use composition and enums; an enum with a `match` is the Rust way to write a sealed class hierarchy.
  - Taking `String`/`Vec<T>` parameters where `&str`/`&[T]` suffice — forces callers to allocate.
- **Holding a std `MutexGuard` across `.await`.** `std::sync::MutexGuard` is `!Send`, so `tokio::spawn` refuses the future with a confusing "future cannot be sent between threads safely" error. Fix: end the lock scope before awaiting — `let v = { state.lock().unwrap().val.clone() }; do_async(v).await;`. Use `tokio::sync::Mutex` only when the guard genuinely must live across an await; otherwise std Mutex + short critical sections is faster and less deadlock-prone.
- **Reading `Send` errors backwards.** "Future is not Send" at a spawn site is rarely about the spawned function itself. Scan the async body for what is *alive across an await*: `Rc`, `RefCell` borrows, `MutexGuard`, raw pointers, `Cell`. Drop it or scope it (`{ ... }`) before the await point. The compiler's "captured value is not Send" note names the culprit — read the note, not just the headline.
- **Cancellation-unsafety.** Any future can be dropped at any await point. In `tokio::select!`, every losing branch is dropped *each iteration*. Concrete bugs: a half-completed two-step write (money moved out, never in); buffered data lost when a read future is dropped mid-protocol; state mutated before an await, never rolled back. Defenses: make await-separated steps idempotent or transactional; prefer documented cancel-safe ops in select arms (tokio docs mark cancel safety per method — `mpsc::Receiver::recv` is cancel-safe; many buffered/framed reads are not); do multi-step work in a spawned task that owns its data, with the select loop only dispatching.
- **Blocking the runtime.** Sync I/O or heavy CPU inside async stalls a worker thread; enough of them stalls everything — the symptom is unrelated requests and timers slowing *together*. Use `tokio::task::spawn_blocking` for sync/CPU work; a rayon pool bridged with a oneshot channel for data-parallel compute. Rule of thumb: >100µs of non-awaiting work in a hot async path deserves a look.
- **`async fn` in traits: dispatch limits.** Native async-fn-in-trait works for generic dispatch but such traits aren't directly usable as `dyn`; for trait objects use the `async-trait` crate (boxes futures) or hand-written `-> Pin<Box<dyn Future + Send + '_>>`. In public library traits, prefer explicit `impl Future + Send` returns so you control the `Send` promise.
- **Unsafe discipline.** Every `unsafe` block carries a `// SAFETY:` comment stating the invariant and why it holds *at this call site*. Wrap unsafe in a minimal safe API; the *module* is the soundness boundary — audit every function that can touch the invariant (e.g., anything that can set `len` on a hand-rolled Vec), not just the marked block. Run **Miri** (`cargo +nightly miri test`) on any crate with nontrivial unsafe — it catches aliasing UB and use-after-free that tests can't. Edition 2024 requires explicit unsafe blocks inside `unsafe fn` — don't blanket-allow the lint.
- **Feature-flag misuse.** Cargo features are unified across the dependency graph, so they must be *additive* — an "exclusive backend" or default-on `std` modeled as competing features breaks when two dependents disagree. Patterns: minimal defaults in libraries; `dep:` syntax for optional dependencies; CI job with `--no-default-features` and one with `--all-features`. In workspaces: shared versions in `[workspace.dependencies]`, inherited via `foo.workspace = true`; remember workspace-wide builds unify features and can mask per-crate breakage — periodically build leaf crates alone.
- **Panic policy drift.** `unwrap()`/`expect()` in libraries turns callers' recoverable situations into aborts. Library code: return `Result`; `expect("invariant: X")` only for provable invariants, with the proof in the message. Binaries: top-level anyhow + `?` everywhere; panics only for bugs.
- **Iterator adapter vs loop borrow fights.** `self.items.iter().map(|i| self.transform(i))` hits E0502 when `transform` takes `&mut self`. Collect first (`let ids: Vec<_> = self.items.iter().map(|i| i.id).collect();` then loop), or restructure per the field-split move. Don't contort into `unsafe` iterator gymnastics.
- **String types confusion.** Parameters: `&str` (accept the world). Storage: `String`. Returns: `String` (or `&str` borrowed from an input with an honest lifetime). `AsRef<str>`/`Into<String>` generics on *ergonomic public APIs* only — they cost compile time and inference clarity; plain `&str` is right for internal code. `Cow<'_, str>` when you sometimes modify — measure before assuming it matters.
- **`impl Trait` capture surprises (edition 2024).** Return-position `impl Trait` now captures all in-scope lifetimes by default; if callers hit "borrowed value does not live long enough" on your returned iterator, add `use<>` (or `use<'a>`) to declare precisely what's captured.
- **Integer overflow semantics split.** Debug builds panic on overflow; release builds wrap silently. Arithmetic on untrusted input needs `checked_*`/`saturating_*`/`wrapping_*` chosen explicitly — the default is a behavior difference between your tests and production.
- **Shadowing `Result` with type aliases across crates.** `type Result<T> = std::result::Result<T, MyError>` is fine within a crate; re-exporting it publicly confuses downstream `?` conversions. Keep aliases crate-internal.

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
pub fn get(k: &str) -> Result<Vec<u8>, StoreError> { todo!() }

// binary crate: context-rich, untyped
use anyhow::Context;
fn main() -> anyhow::Result<()> {
    let cfg = std::fs::read_to_string("app.toml").context("reading app.toml")?;
    let v = store::get("user:1").context("loading user")?;   // StoreError -> anyhow via ?
    // ...
    Ok(())
}
```

**Cancel-safe select loop shape (dispatcher owns nothing mid-flight):**
```rust
use tokio_util::task::TaskTracker;

let tracker = TaskTracker::new();
loop {
    tokio::select! {
        biased;                              // poll shutdown first, deterministically
        _ = shutdown.changed() => break,
        Some(job) = rx.recv() => {           // mpsc recv is documented cancel-safe
            // Do NOT await multi-step work here: a shutdown would drop it midway.
            tracker.spawn(handle(job));      // the task owns `job` to completion
        }
    }
}
tracker.close();
tracker.wait().await;                        // drain in-flight work before exit
```

**Ownership restructuring instead of RefCell (the field-split move):**
```rust
// BEFORE: fn tick(&mut self) { for r in &self.rules { self.apply(r); } }  // E0502
// AFTER: split so borrows are disjoint per field
struct Sim { rules: Vec<Rule>, engine: Engine }
impl Sim {
    fn tick(&mut self) {
        for r in &self.rules {          // immutable borrow of self.rules
            self.engine.apply(r);       // mutable borrow of self.engine — disjoint, compiles
        }
    }
}
```

**Workspace + feature layout that survives growth:**
```toml
# Cargo.toml (root)
[workspace]
members = ["crates/core", "crates/store", "crates/api", "crates/cli"]
resolver = "3"                       # edition-2024 default; declare it explicitly at the root

[workspace.dependencies]             # single source of version truth
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
thiserror = "2"
anyhow = "1"

# crates/store/Cargo.toml
[dependencies]
serde.workspace = true
thiserror.workspace = true
core = { path = "../core" }

[features]
default = []                         # libraries: minimal defaults
postgres = ["dep:sqlx"]              # additive opt-in via dep: syntax
[dependencies.sqlx]
version = "0.8"
optional = true
```
Dependency direction: `cli`/`api` → `store` → `core`; traits live in `core` so higher crates can implement them (coherence). CI runs `--no-default-features` and `--all-features` jobs to catch feature-unification masking.

## Verification and stopping rule

Before presenting Rust code or advice:
1. Assume `cargo clippy --all-targets -- -D warnings` runs — would it pass? Clippy encodes most of this skill's idioms mechanically.
2. Every `clone`, `Rc<RefCell<_>>`, `unwrap`, and `unsafe` must have a one-line justification you can state; if you can't, restructure before presenting.
3. For async code, walk each `await` twice: "what happens if the future is dropped exactly here?" and "what is alive across this point that isn't `Send`?"
4. For unsafe, the SAFETY comment must cite a *checkable* invariant, and recommend Miri in CI.
5. Never claim performance without criterion/`cargo bench` numbers; monomorphization vs vtable dispatch differences are usually noise outside hot loops.
6. Version/edition-gate advice: async closures and `use<...>` capture syntax need edition 2024 / recent compilers; check the crate's `edition` and MSRV before recommending them.

Stopping rule: done when ownership reads as a tree, public APIs take borrows and return owned (or fully owned types at thread/async seams), errors are typed exactly where callers match and anyhow above, and every goroutine-like spawned task has an owner awaiting it. Chasing zero-clone purity in cold paths, or generic-ifying code with one concrete user, is waste.
