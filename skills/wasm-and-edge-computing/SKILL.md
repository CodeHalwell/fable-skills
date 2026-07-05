---
name: wasm-and-edge-computing
description: Load when working with WebAssembly (browser or server-side — Rust/Go/C to wasm, WASI, the component model, plugin sandboxing) or edge compute platforms (Cloudflare Workers class — edge routing, KV/Durable Objects/D1 state, cold starts, CPU limits), or deciding whether wasm/edge is the right tool at all.
---

# WebAssembly and Edge Computing

## Core mental model

- **Wasm is sandboxed portable compute, not "fast JavaScript."** Its two real value propositions: (1) near-native, *predictable* CPU performance (no JIT warmup cliffs, no GC pauses for linear-memory languages), and (2) a capability-based sandbox — a wasm module can touch nothing you don't explicitly hand it. Many wins attributed to speed are actually wins from portability (one binary, every platform) or sandboxing (run untrusted code in-process).
- **The JS boundary is the real performance story in browsers.** Compute inside wasm is fast; *crossing* into JS costs — especially strings and objects, which must be copied/re-encoded through linear memory (wasm has no direct DOM access; every DOM touch is a JS call). A workload that crosses the boundary per-element loses to plain JS; one that crosses per-*batch* wins big. Chatty interface = wasm loses; chunky interface = wasm wins. Design the boundary first, then the module.
- **Modern JS engines are very fast.** For typical app logic — small functions over JS objects, string munging, DOM-driven UI — a well-JIT'd JS version is within striking distance of wasm and free of boundary costs. Wasm needs sustained numeric/byte-level compute over data that can *live* in linear memory to justify itself.
- **Edge computing's binding constraint is data locality, not compute.** Running code in 300 cities is easy; your database lives in one region. An edge function that makes three round-trips to a us-east-1 Postgres is slower than a us-east-1 server making local queries. What belongs at the edge is what can be decided with *local or replicated* state.
- **Edge platforms sell isolation economics.** V8 isolates (Cloudflare Workers, Deno Deploy) and wasm sandboxes (Fastly Compute) start in microseconds-to-milliseconds and pack thousands per machine — that's why edge is cheap and cold-start-free relative to container/microVM serverless (Lambda class). The price: constrained runtimes (no arbitrary native code, CPU-time budgets, no long-lived local state).

## When wasm actually wins — the decision chain

Ask in order:
1. **Is the hot path sustained CPU over bulk data?** Codecs (image/video/audio transcode), compression, crypto, physics/simulation, geometry, parsers over big buffers, ML inference kernels, spreadsheet-grade recalculation → wasm's home turf. If the profile is dominated by DOM, I/O, or scattered small object work → stay in JS; wasm will add complexity and a boundary tax for nothing.
2. **Is there an existing C/C++/Rust library that is the asset?** SQLite, FFmpeg, libvips, Skia, a proprietary engine — compiling proven code to wasm beats reimplementing in JS by an order of magnitude in effort and correctness. This is the most common *legitimate* browser-wasm reason in practice.
3. **Do you need to run untrusted/third-party code?** Plugin systems (Figma-style), user-defined functions in a SaaS, per-tenant extensions → wasm's sandbox + fuel/epoch-based CPU metering (Wasmtime supports both) is the engineering-grade answer; V8 isolates are the alternative if plugins are JS anyway.
4. **Do you need one artifact across browser/server/edge/embedded?** Portable business logic, shared validation, cross-platform SDK cores → wasm as the distribution format.
If none of the four: you don't need wasm. "We rewrote our React app's utils in Rust" is a negative-ROI story with rare exceptions — say so.

## The toolchain map (verified as of mid-2026)

- **Rust**: the mature path and dominant wasm source language. Browser: `wasm32-unknown-unknown` + `wasm-bindgen` (v0.2.x line, actively maintained) + `wasm-pack`. Server/WASI: `wasm32-wasip1` and `wasm32-wasip2` targets — wasip2 has been tier-2 since Rust 1.82 and builds **components** directly via `cargo build --target wasm32-wasip2`; `cargo-component` is being superseded by that direct path, with `wit-bindgen` for custom interfaces.
- **Go**: mainline Go compiles to wasm but ships a large runtime (MB-scale binaries, GC included) — fine for browser apps that amortize it, poor for edge/plugin use. **TinyGo** produces small binaries and supports WASI/components, at the cost of incomplete stdlib/reflection support — test your actual dependencies early. (Known sharp edge: TinyGo component builds can silently fall back to the default `wasi:cli/run` world if your WIT world isn't found — verify the produced component's world, don't assume.)
- **C/C++**: Emscripten for browser (mature, huge porting ecosystem, POSIX shims); wasi-sdk for clean server-side WASI.
- **JS/TS *inside* wasm**: possible via engines like StarlingMonkey/porffor-class tooling and `jco` for componentizing — used when a wasm host must run JS, not for speed.
- **Standards state (as of mid-2026)**: **WASI 0.2** (component model-based) is the stable baseline with an ecosystem around it; **WASI 0.3.0 shipped June 2026**, adding native async to the component model (`async func`, `stream<T>`, `future<T>`; `wasi:io` absorbed into the canonical ABI), supported in Wasmtime 43+ and jco — new server-side designs should plan for 0.3-style async but can ship on 0.2 today. The old non-component `wasip1` remains what much deployed tooling emits; know which world you're in, because "WASI support" claims differ by a whole ABI. Browser-side, wasm GC and threads/SIMD are broadly available in modern engines; the component model is a *server-side* story so far.
- **Runtimes**: Wasmtime (reference-quality, component-model-first), WasmEdge/Wasmer (alternatives), workerd (Cloudflare's runtime, V8-based — runs wasm *via* V8).

## Edge platform landscape and architecture judgment

Platform classes (as of 2026): **V8 isolates** — Cloudflare Workers (the most mature ecosystem), Deno Deploy; **wasm-native** — Fastly Compute, wasmCloud/Spin-class for self-hosted; **microVMs/containers at edge POPs** — Lambda@Edge/CloudFront Functions, Fly.io (full VMs near users — different tradeoff: real processes, higher floor cost). Notably, some platforms have *retreated* from edge-first execution for general app code back toward regional compute (Vercel's shift of functions to regional "fluid" compute is the prominent example) — evidence that the data-locality constraint, not enthusiasm, decides what runs at the edge.

**What belongs at the edge** (decidable with local/replicated state, latency-sensitive, per-request): routing and load steering, authN token *verification* (JWT signature checks — no DB needed), A/B and feature-flag assignment (config replicated via KV), personalization from cookie/header/geo, redirects/rewrites, caching logic, bot filtering, WAF-adjacent checks, API composition/response shaping.

**What doesn't**: multi-statement DB transactions against a regional primary (each statement pays the round trip — the transaction that took 5 ms app-side takes 300 ms edge-side), heavy write workflows, anything needing large memory/long CPU, fan-out aggregations over regional services. Rule: **move the decision to the edge, keep the transaction near the data.** If you find yourself proxying every edge request to one region anyway, the edge tier is adding a hop, not subtracting latency.

**Edge state primitives** (Cloudflare's taxonomy, mirrored by competitors — as of 2026):
- **KV**: global, replicated, *eventually consistent* — writes can take up to ~60 s to propagate globally. For config, flags, sessions-that-tolerate-staleness. Never for counters or read-modify-write.
- **Durable Objects**: single-instance-per-key actor with strongly consistent storage (SQLite-backed, ~10 GB per object) — *the* primitive for coordination: counters, rate limiters, WebSocket rooms, per-entity locks. Consistency comes from single placement, which means cross-region callers pay the trip to wherever the object lives.
- **D1**: managed SQLite (10 GB/DB cap), regional primary with read replication — relational data at edge-adjacent latency for reads; it is not a distributed SQL engine.
- **R2/blob + cache**: bulk data. General rule: reads scale globally, *authoritative writes serialize somewhere* — pick the primitive by asking where the write must be consistent.

**Choosing a state primitive — the reasoning chain**: *Does the write need to be read-your-write consistent by the next request?* No → KV (config, flags, cached derivations). Yes → *is the consistency scope a single logical entity* (one room, one counter, one user session)? → Durable Object keyed on that entity (single-writer actor; consistency by placement). *Is it relational queries over shared data?* → D1 or, more often, your existing regional database reached via a connection-pooling gateway (Hyperdrive-class) — be honest that this is regional data with an edge cache, not "edge state." *Is it large blobs?* → R2 + cache API. If you can't state which primitive owns each piece of state and why, the architecture isn't done.

**Limits arithmetic** (Cloudflare current, as of 2026 — other platforms differ in numbers, not shape): free tier 10 ms CPU/request and 100K requests/day; paid tier CPU default 30 s max but you should budget single-digit ms for request-path work; 128 MB memory per isolate. Key mental split: **CPU time ≠ wall time** — awaiting a fetch costs no CPU budget, so I/O-bound orchestration is nearly free while a 50 ms JSON-parse of a huge body blows the budget. Estimate: parsing/serializing ~1 MB of JSON ≈ several ms of CPU — a 10 ms budget means the free tier cannot afford megabyte payload transforms per request.

**Cold-start arithmetic**: V8 isolates spin up in ~5 ms or less (often pre-warmed to ~0); wasm instances similar once compiled; container/microVM serverless is 100 ms–seconds cold. This is why "put the auth check at the edge" works: the platform's isolation floor is below your latency noise. But per-isolate caches start empty at every POP — a cache with a 1% hit rate per POP × 300 POPs is not a cache; centralize caching in the platform's cache API or KV, and treat isolate memory as request-scoped with occasional luck.

## Local-dev and testing realities

- `wrangler dev` runs the real workerd runtime locally (high fidelity for JS semantics and bindings APIs), but *not* the distributed behavior: KV propagation is instant locally, Durable Objects are all "nearby," CPU limits are not enforced by your laptop. The bugs that matter — consistency windows, cross-POP placement latency, budget overruns — only reproduce deployed.
- Test pyramid that works: unit tests against binding mocks (`vitest` + platform test pools) → integration on a deployed staging zone (real KV latencies, real limits) → a small always-on synthetic probe hitting production from multiple regions, because edge behavior is region-dependent by construction.
- For wasm modules: test the core logic natively (Rust `cargo test` — fast, debuggable), test the *boundary* in the actual host (browser test or Wasmtime harness); most wasm bugs live in the boundary glue, not the kernel. Debugging story (as of 2026): DWARF-based source-level debugging works in Chrome DevTools for browser wasm and improves steadily server-side, but `println!`-style logging through a host function remains the workhorse — wire a logging import from day one.

## How an expert thinks through it: "should this image pipeline move to wasm at the edge?"

SaaS resizes/re-encodes user images (p50 800 KB) in a regional Node service; team proposes "wasm on Workers for latency." Internal monologue: *Two separate proposals hiding in one sentence — wasm (execution tech) and edge (placement). Evaluate independently. Placement first: where are the images? Origin bucket in one region; output goes to a CDN anyway. Moving compute to 300 POPs doesn't move the source bytes — each edge invocation would fetch the original across the world, transform, and re-upload. Latency win: near zero (CDN already serves cached derivatives); egress and complexity: up. Also check the budget: image decode+encode of an 800 KB JPEG is tens-to-hundreds of ms of CPU — far beyond comfortable request-path budgets on an isolate platform, and 128 MB memory is tight for large images. Edge placement: rejected on data locality and CPU arithmetic.* *Now wasm as tech, staying regional: the service currently shells out to ImageMagick — process-spawn overhead and a long CVE history on untrusted input. Compiling libvips-class processing to wasm and running it in-process under Wasmtime with fuel metering gives sandboxing of a notoriously exploitable parser + removes spawn overhead. That's the capability win, not a latency win.* *What stays at the edge? The 5-line decision: parse the URL's transform params, check the derivative cache, only cache-misses hit the regional transformer.* Final architecture: edge does routing/cache/auth (its job), regional wasm-sandboxed workers do the compute (their job). *Also rejected: rewriting the pipeline in Rust-native without wasm — viable, loses the sandbox; and client-side wasm transforms — uploads user CPU, but originals must be canonicalized server-side anyway for the CDN.* Stopping rule: decision is done when each component sits where its data is and the CPU arithmetic fits the platform budget with ≥5× headroom.

## Failure modes & pitfalls

- **Per-element boundary crossings**: calling a wasm function inside a JS loop over 100K items, or wasm calling back into JS per item. The copies/transitions dominate; the "optimized" version is slower than JS. Correction: pass one TypedArray / pointer+length per batch; design APIs as bulk operations over linear memory.
- **String-heavy interfaces**: every JS↔wasm string is a UTF-16↔UTF-8 re-encode + copy. A wasm "text processor" called per-line can spend most of its time encoding. Correction: move whole documents across once; return offsets/indices instead of substrings where possible.
- **Benchmarking wasm compute while ignoring instantiation + boundary + download**: the module wins the kernel benchmark and loses the user-visible one. Correction: measure end-to-end; use `WebAssembly.instantiateStreaming` (compiles during download); lazy-load the module off the critical path.
- **Assuming "WASI support" means one thing**: a toolchain emitting `wasip1` core modules won't run in a host expecting `wasip2` components (and vice versa) — the errors are opaque. Correction: state the target explicitly end-to-end; `wasm-tools component wit mod.wasm` to inspect what you actually built.
- **Mainline Go for size-sensitive wasm**: multi-MB binaries and GC where you needed 100 KB. Correction: TinyGo, or Rust if TinyGo's stdlib gaps block you — check reflection-heavy deps (encoding/json) early.
- **Trusting the sandbox for what it doesn't cover**: wasm confines memory and capabilities, not CPU or timing side-channels. An untrusted plugin can still spin forever or mine crypto. Correction: fuel or epoch interruption (Wasmtime) for CPU limits; cap memory at instantiation; treat the *host functions you expose* as your real attack surface — the sandbox is only as tight as its imports.
- **Read-modify-write on eventually consistent KV** (view counters on Workers KV): last-write-wins across POPs silently drops updates. Correction: Durable Object per counter key (single-writer serialization), or accept approximate counts explicitly.
- **Putting a chatty DB transaction at the edge**: 5 sequential queries × 150 ms RTT = a 750 ms "edge-accelerated" endpoint. Correction: move the whole transaction into one regional call (RPC/stored procedure/regional function) and let the edge call it once; or replicate the read model outward.
- **Global in-memory state in a Worker as if it were a server**: isolates are per-POP, many-instanced, and evicted at will — an in-memory cache "works" in dev (one instance) and is incoherent in prod. Correction: in-memory only as best-effort per-isolate cache; correctness state goes to KV/DO/D1 per the consistency table.
- **Blowing CPU budget on payload transforms**: streaming a response through `JSON.parse`/`JSON.stringify` of MB-scale bodies on a 10 ms budget. Correction: stream bytes untouched when possible (`resp.body` passthrough); transform at origin; upgrade tier consciously if transform-at-edge is genuinely required.
- **Local-dev overconfidence**: `wrangler dev`-class simulators (workerd locally) are good but not identical — production propagation delays (KV), colo-dependent behavior, and real limits don't reproduce locally. Correction: staging on the real platform; test eventual-consistency windows explicitly; load-test CPU limits with production-shaped payloads, because local machines won't enforce the budget the platform will.

## Worked micro-example: batch-oriented JS↔wasm boundary (Rust)

```rust
// lib.rs — one crossing for the whole buffer, not one per pixel.
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn grayscale(rgba: &mut [u8]) {           // &mut [u8] maps to a Uint8Array view
    for px in rgba.chunks_exact_mut(4) {
        let y = (0.299 * px[0] as f32 + 0.587 * px[1] as f32 + 0.114 * px[2] as f32) as u8;
        px[0] = y; px[1] = y; px[2] = y;      // alpha untouched
    }
}
```

```js
// Build: wasm-pack build --target web --release
import init, { grayscale } from "./pkg/imgfx.js";
await init();                                  // instantiateStreaming under the hood
const data = ctx.getImageData(0, 0, w, h);     // one copy out of the canvas
grayscale(data.data);                          // one boundary crossing, in-place in linear memory
ctx.putImageData(data, 0, 0);                  // one copy back
```

The anti-pattern this replaces: a `grayscale_pixel(r, g, b)` export called w×h times — typically slower than pure JS. Server-side equivalent: compile the same crate to `wasm32-wasip2` and host under Wasmtime with `store.set_fuel(...)` (or epoch deadlines) when the input is untrusted.

## Worked micro-example: sandboxed plugin host with CPU metering (Rust + Wasmtime)

```rust
use wasmtime::{Config, Engine, Module, Store, Linker};

let mut cfg = Config::new();
cfg.consume_fuel(true);                       // deterministic CPU metering
let engine = Engine::new(&cfg)?;
let module = Module::from_file(&engine, "plugin.wasm")?;   // untrusted third-party code

let mut store = Store::new(&engine, HostState::default());
store.set_fuel(50_000_000)?;                  // hard CPU budget for this invocation
store.limiter(|s| &mut s.limits);             // StoreLimits: cap memory growth (e.g. 64 MiB)

let mut linker = Linker::new(&engine);
// The imports ARE the attack surface — expose the minimum, audited:
linker.func_wrap("host", "log", |msg_ptr: i32, len: i32| { /* bounded copy + log */ })?;
// Deliberately NOT linked: filesystem, network, clocks (unless the plugin contract needs them).

let instance = linker.instantiate(&mut store, &module)?;
let run = instance.get_typed_func::<(i32, i32), i32>(&mut store, "process")?;
match run.call(&mut store, (input_ptr, input_len)) {
    Ok(rc) => handle(rc),
    Err(trap) => quarantine_plugin(trap),     // out-of-fuel / OOM traps land here — plugin
}                                             // dies, host thread is fine. That's the product.
```

Fuel (per-instruction accounting, deterministic, slight overhead) vs. epochs (wall-clock deadline via `epoch_interruption`, near-zero overhead, non-deterministic): choose fuel when reproducible budgets matter (billing plugins per compute), epochs when you just need "nothing runs longer than 100 ms." Same sandboxing logic applies whether plugins come from `wasm32-wasip2` Rust, TinyGo, or AssemblyScript — the host code doesn't change, which is the component model's point.

## Verification / self-check

1. **Boundary audit**: count JS↔wasm crossings per user operation. O(1) per operation = healthy; O(n) in data size = redesign before shipping.
2. **Honest benchmark**: compare against a *tuned* JS implementation (TypedArrays, no allocation in the loop), end-to-end including load and copies — not against naive JS, and not kernel-only. If wasm wins by <2×, the complexity usually isn't paid for.
3. **Artifact inspection**: check binary size (`wasm-opt -O` applied? unexpected MBs usually mean panicking formatters or a dragged-in runtime); for components, verify the WIT world matches the host.
4. **Edge deployment**: verify CPU-ms p99 against the plan's budget with ≥5× headroom on production-shaped payloads; confirm every piece of state maps to a primitive whose consistency you can state in one sentence; kill-test — does the app degrade correctly when KV serves 50 s-stale data?
5. **Sandbox claims**: if you say "untrusted code is safe," show the fuel/epoch limit, the memory cap, and the audited import list — all three, or the claim is wrong.

Stopping rule: optimization ends when the boundary is batch-shaped, the honest benchmark shows the win you claimed, and platform budgets have headroom. Squeezing further wasm micro-optimizations past that point loses to shipping.
