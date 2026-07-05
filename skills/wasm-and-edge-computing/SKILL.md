---
name: wasm-and-edge-computing
description: Load when working with WebAssembly (browser or server-side — Rust/Go/C to wasm, WASI, the component model, plugin sandboxing) or edge compute platforms (Cloudflare Workers class — edge routing, KV/Durable Objects/D1 state, cold starts, CPU limits), or deciding whether wasm/edge is the right tool at all.
---

# WebAssembly and Edge Computing

Frontier models already hold the fundamentals cold: boundary-crossing economics, when-wasm-wins criteria, fuel-vs-epoch metering, KV/DO/D1 consistency and limits, linear-memory ratcheting, COOP/COEP for threads. This sheet keeps the decision tables for review completeness, the 2026 toolchain/standards anchors (where cold knowledge is stale), and the field-only pitfalls.

## Anchors (one line each)

1. Wasm = sandboxed portable *predictable* compute, not "fast JS" — most wins attributed to speed are portability or sandboxing wins.
2. The JS boundary is the browser performance story: chatty interface = wasm loses; chunky (batch, pointer+length, data resident in linear memory) = wasm wins. Design the boundary first.
3. Wasm earns its place on exactly four grounds: sustained CPU over bulk bytes; an existing C/C++/Rust library as the asset; untrusted-code sandboxing; one artifact across platforms. None of the four → don't.
4. Edge's binding constraint is data locality: move the *decision* to the edge, keep the *transaction* near the data. If every edge request proxies to one region, the edge tier adds a hop.
5. Isolation economics: V8 isolates/wasm sandboxes start in ~5 ms or less and pack thousands per machine — that's the product; the price is constrained runtimes and CPU-time budgets.

Placement table (kept for completeness):

| Workload | Verdict |
|---|---|
| Image/video/audio transcode in browser | wasm, in a Worker |
| Form validation, app state, DOM-driven UI | JS |
| SQLite/DuckDB in browser | wasm (official builds) |
| Per-tenant scripts in a SaaS backend | wasm + Wasmtime fuel/epoch |
| JWT verify / AB assignment / redirects | edge, plain JS |
| Checkout transaction | regional service |
| Crypto beyond WebCrypto | wasm (constant-time survives; JS JIT timing untrustworthy) |

## Toolchain and standards anchors (verified mid-2026 — this is where cold models are behind)

- Rust: browser = `wasm32-unknown-unknown` + wasm-bindgen/wasm-pack; server = `wasm32-wasip2` builds **components directly** (`cargo build --target wasm32-wasip2`, tier-2 since Rust 1.82); `cargo-component` is superseded by that direct path; `wit-bindgen` for custom interfaces.
- **WASI 0.3.0 shipped June 2026** — native async in the component model (`async func`, `stream<T>`, `future<T>`; `wasi:io` absorbed into the canonical ABI), supported in **Wasmtime 43+** and jco. Cold models still say "Preview 3 in progress." New server-side designs: plan for 0.3-style async, ship on 0.2 today. "WASI support" claims differ by a whole ABI (wasip1 core modules vs wasip2 components) — state the target end-to-end; `wasm-tools component wit mod.wasm` to inspect what you actually built.
- Go: mainline = MB-scale binaries + GC (browser-only viability); TinyGo for edge/plugins — **sharp edge: TinyGo component builds can silently fall back to the default `wasi:cli/run` world when your WIT world isn't found; verify the produced component's world.** Check reflection-heavy deps (encoding/json) early.
- Evidence line worth citing in placement debates: some platforms have *retreated* from edge-first execution for general app code back to regional compute (Vercel's "fluid" regional shift) — data locality, not enthusiasm, decides.
- Cloudflare numbers (2026, shape generalizes): free 10 ms CPU/request; paid budget single-digit ms for request-path work; 128 MB/isolate; CPU ≠ wall (awaited fetches are free) — ~1 MB JSON parse ≈ several ms, so free tier can't afford MB-payload transforms per request. KV eventually consistent (~60 s propagation, no CAS); DO = single-placement actor, SQLite-backed ~10 GB; D1 = regional-primary SQLite (10 GB), not distributed SQL. State rule: reads scale globally, authoritative writes serialize somewhere — pick the primitive by where the write must be consistent, and shard DOs by natural key (a singleton DO is a worldwide bottleneck).

## Field pitfalls a cold model doesn't volunteer

- **Compiling the module per invocation server-side**: `Module::from_file`/`Module::new` runs a full compile — per request that turns a 2 ms plugin call into 200 ms. Compile once, cache the `Module` (or ship precompiled artifacts); create only the cheap `Store`/instance per request.
- **Missing host capabilities discovered at runtime**: wasm has no ambient clock/entropy/fs; `getrandom`-class crates trap or return constants unless the host wires the import. Enumerate imports at build time (`wasm-tools`) and treat unexpected imports as a *build* error.
- **The sandbox is only as tight as its imports** — fuel/epoch caps CPU and `StoreLimits` caps memory, but the host functions you expose are the real attack surface. "Untrusted code is safe" requires all three shown: CPU limit, memory cap, audited import list.
- Per-isolate caches at 300 POPs are not a cache (1% hit rate each) — centralize in the cache API/KV; isolate memory is request-scoped with occasional luck.
- `wrangler dev` runs real workerd but not distributed reality: KV propagation instant locally, DOs all "nearby," CPU limits unenforced — consistency windows and budget overruns only reproduce deployed. Staging zone + multi-region synthetic probes.
- Binary bloat: `wasm-opt -O` (10–30% routine), Rust `panic = "abort"`, no `format!` in hot paths (panicking formatters ≈ 100+ KB); `twiggy` when size surprises you.
- Honest benchmark rule: compare against *tuned* JS (TypedArrays, no allocation in loop), end-to-end including load/instantiation/copies. Win <2× → the complexity isn't paid for.

## Worked micro-example: batch boundary + metered plugin host (merged essentials)

```rust
// One crossing per buffer, not per pixel. &mut [u8] maps to a Uint8Array view.
#[wasm_bindgen]
pub fn grayscale(rgba: &mut [u8]) {
    for px in rgba.chunks_exact_mut(4) {
        let y = (0.299*px[0] as f32 + 0.587*px[1] as f32 + 0.114*px[2] as f32) as u8;
        px[0] = y; px[1] = y; px[2] = y;
    }
}
```

```rust
// Untrusted plugin host (Wasmtime): the three sandbox proofs in one place.
let mut cfg = Config::new();
cfg.consume_fuel(true);                       // fuel = deterministic budgets (billing/replay);
let engine = Engine::new(&cfg)?;              // epochs = cheap wall-clock deadlines
let module = Module::from_file(&engine, "plugin.wasm")?;   // compile ONCE, cache
let mut store = Store::new(&engine, HostState::default());
store.set_fuel(50_000_000)?;                  // 1. CPU budget
store.limiter(|s| &mut s.limits);             // 2. memory cap
let mut linker = Linker::new(&engine);        // 3. audited imports — the attack surface
linker.func_wrap("host", "log", |ptr: i32, len: i32| { /* bounded copy */ })?;
// Deliberately NOT linked: fs, network, clocks.
// Out-of-fuel/OOM traps kill the plugin, not the host thread. That's the product.
```

## Verification / self-check

1. Boundary audit: crossings per user operation O(1), not O(n) in data size.
2. Honest benchmark (above) shows the claimed win.
3. Artifact inspected: size sane, component's WIT world matches the host.
4. Edge: CPU-ms p99 ≥5× headroom on production-shaped payloads; every piece of state names its primitive and consistency in one sentence; degrades correctly on 50 s-stale KV.
5. Sandbox claims show fuel/epoch + memory cap + import list.

Stopping rule: batch-shaped boundary, honest benchmark won, platform budgets with headroom — further wasm micro-optimization loses to shipping.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 hard delta.
- Sharpened: WASI 0.3.0 actually shipped June 2026 (Wasmtime 43+, jco) — cold models hedge "Preview 3 in progress"; wasip2-direct build having superseded cargo-component stated as settled.
- Opus cold nails: boundary economics, four-grounds decision, fuel-vs-epoch tradeoff, KV/DO/D1 table incl. ~60 s propagation and lost-update counters, CPU-vs-wall + JSON-parse arithmetic, linear-memory ratcheting, COOP/COEP. Kept field-only pitfalls it doesn't volunteer: per-invocation Module compile, import-enumeration-as-build-gate, TinyGo silent world fallback, Vercel regional-retreat evidence, per-isolate cache mirage.
