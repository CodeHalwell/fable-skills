---
name: rust-development
description: Loads expert Rust judgment for ownership-driven design, lifetimes, smart-pointer and error-handling architecture, async pitfalls, trait design, and unsafe discipline. Use when writing or reviewing Rust, resolving borrow-checker fights, choosing Box/Rc/Arc/RefCell/Mutex, structuring errors with thiserror/anyhow, or debugging Send/lifetime errors in async code.
---

# Rust Development

## Core mental model (anchors)

1. A borrow error is a design review finding — restructure ownership (split structs, narrow scopes, pass indices, take by value), don't reach for `clone()`/`Rc<RefCell<_>>`/`unsafe`. Each escape hatch trades a static guarantee for a dynamic one (RefCell → runtime panic, Mutex → deadlock, unsafe → UB).
2. Rust programs are ownership trees; graphs need deliberate machinery (indices, arenas, `Weak`).
3. Dropping a future is cancellation, at any await point — this single fact explains most async production bugs.

## Current state (verified July 2026)

- Stable Rust is the **1.96.x line** (1.96.1, June 2026) — cold estimates run ~4 minors low; check before citing "requires Rust ≥N" claims. Edition 2024 (since 1.85) is the production default: async closures (`async || {}`) — a 2024-edition feature cold answers tend to omit — `use<...>` capture syntax for RPIT, `unsafe_op_in_unsafe_fn` by default, resolver v3.
- tokio is the default runtime **and has LTS lines (e.g. 1.47/1.51, multi-year windows)** — pin one for long-lived services. async-std is discontinued (treat it in a dependency tree as a migration flag); smol survives in the lightweight niche. axum / thiserror+anyhow / serde / clap are the consolidated defaults.
- Libraries: don't hard-wire a runtime when feasible — accept `impl Future`, feature-gate tokio integration.

## Decision anchors (compressed — cold answers reproduce the full chains)

- Smart pointers, in order: `T`/`&T`/`&mut T` → `Box` → `Rc`/`Arc` → interior mutability (`RefCell`/`Mutex`/atomics) → `Weak` for cycles. Reaching interior mutability in *application logic* (not caches/registries/GUI) is a smell; the usual fix is channels or a single mutating owner.
- Lifetimes: a struct holding `&'a T` infects every user; reserve borrowing structs for short-lived views (parsers, guards). `T: 'static` means "no non-static borrows," not "lives forever" — most `tokio::spawn` `'static` errors are fixed by moving owned data in. **Escalation stop: a lifetime puzzle exceeding ~10 minutes is the design saying "own the data"** — a clone at a boundary costs nanoseconds; a `'a` in a public API costs every downstream user forever.
- Errors: thiserror where callers branch on variants; anyhow (+ `.context`) in binaries/orchestration; conversion once at the boundary via `?`. Anti-patterns: anyhow in a published API; one 30-variant crate-wide enum nobody matches (that's anyhow with extra steps); `Box<dyn Error>` in new code.
- Generics for hot paths and simple bounds; `dyn` for registries/heterogeneity/compile-time relief. Coherence: define traits in low-level workspace crates; newtype-wrap foreign types.

## The two diagnosis priors

1. **"cannot borrow X because Y is borrowed" across a method call** → a method takes `&mut self` while needing only some fields. The checker splits borrows per *field*, so split the struct (`self.engine.apply(&self.rules[i])` compiles) or make it a free function taking exactly the fields used. Cloning to appease compiles but papers over the finding.
2. **"future cannot be sent between threads safely"** → read the compiler note naming the value alive across an await (MutexGuard, Rc, RefCell borrow). Fix is almost always "make the non-Send thing die before the await" (scope it in `{}`), not "find a Send version." `tokio::sync::Mutex` only when the lock must genuinely span an await — otherwise it legitimizes holding locks across I/O.

## Pitfalls checklist

Baseline one-liners (kept for completeness): clone-to-appease; GC-refugee patterns (Rc<RefCell> soups → arenas/indices; `&str` in long-lived structs → `String`; inheritance-simulation → enums + match); std MutexGuard across `.await`; select! drops losing branches every iteration — multi-step work belongs in a spawned task, cancel safety is documented per method in tokio docs; `spawn_blocking` for sync/CPU work (>100µs non-awaiting work in a hot async path deserves a look); features must be additive, `dep:` syntax, CI with `--no-default-features` and `--all-features` (plus `cargo hack --feature-powerset` on small matrices); debug panics vs release wraps — `checked_*`/`saturating_*` on untrusted input; `unwrap` policy (libraries return Result; `expect("invariant: X")` with the proof); SAFETY comments + Miri, and the soundness audit boundary is the *module* (every safe fn that can touch the invariant), not the marked block.

Expanded — where cold answers are thin:

- **Async-fn-in-trait + dyn dispatch:** native AFIT traits aren't dyn-compatible; for trait objects use `async-trait` (boxes futures) or hand-written `-> Pin<Box<dyn Future + Send + '_>>`. In *public library* traits prefer explicit `impl Future + Send` returns so you control the Send promise rather than `trait_variant` retrofits.
- **`impl Trait` capture (edition 2024):** RPIT now captures all in-scope lifetimes by default; callers hitting "borrowed value does not live long enough" on your returned iterator need you to add `use<>`/`use<'a>` to declare precise capture.
- **Workspace feature unification masks per-crate breakage** — workspace-wide builds unify features, so a leaf crate that doesn't compile alone can hide for months; periodically build leaf crates in isolation.
- **The cancel-safe dispatcher shape** (memorize; cold answers describe the principle but botch the drain):

```rust
let tracker = tokio_util::task::TaskTracker::new();
loop {
    tokio::select! {
        biased;                          // poll shutdown first, deterministically
        _ = shutdown.changed() => break,
        Some(job) = rx.recv() => {       // mpsc recv is documented cancel-safe
            tracker.spawn(handle(job));  // task owns `job` to completion; NEVER await multi-step work in the arm
        }
    }
}
tracker.close();
tracker.wait().await;                    // drain in-flight work before exit
```

- Public type-alias hygiene: keep `type Result<T> = ...` crate-internal; re-exporting it confuses downstream `?` conversions.

## Verification and stopping rule

1. Assume `cargo clippy --all-targets -- -D warnings` runs — would it pass?
2. Every `clone`/`Rc<RefCell<_>>`/`unwrap`/`unsafe` has a one-line justification you can state.
3. Walk each `await` twice: "what if the future is dropped exactly here?" and "what is alive across this point that isn't Send?"
4. Edition/MSRV-gate advice: async closures and `use<...>` need edition 2024; check `edition` before recommending.
5. No performance claims without criterion numbers; monomorphization-vs-vtable differences are usually noise outside hot loops.

Done when ownership reads as a tree, APIs take borrows and return owned, errors are typed exactly where callers match, and every spawned task has an owner awaiting it. Zero-clone purity in cold paths is waste.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta beyond version currency.
- Opus cold nails: E0502 field-split, Send-error triage incl. tokio::sync::Mutex nuance, thiserror/anyhow rule, select! cancel safety + per-method docs, RPIT `use<>`, additive features + cargo-hack, module-boundary unsafe auditing. Skill restructured to a correction sheet.
- Remaining value: exact version currency (cold answers cite ~1.91 when stable is 1.96.x; async closures omitted from edition-2024 lists), tokio LTS lines, the ~10-minute lifetime escalation stop (cold says 15–30min), TaskTracker drain shape, workspace feature-unification masking.
