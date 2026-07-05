---
name: realtime-web
description: Load when building realtime or collaborative web features — choosing transports (SSE vs WebSocket vs WebRTC vs polling), reconnection/resume logic, live sync architectures (CRDT/OT/server-authoritative), presence, message ordering and idempotency, fan-out scaling, or offline-first sync engines.
---

# Realtime Web

## Core mental model (anchors)

1. Realtime is a distributed-systems problem wearing a UI costume: ordering, idempotency, reconnection, conflict resolution — not the socket API.
2. Downstream and upstream are separate decisions; most "realtime" is 95% server→client, and client→server can stay plain HTTP POST (keeps auth/retries/LB/observability boring).
3. The connection is a cache of a subscription: model client state as snapshot + resume cursor, so cold load, reconnect, and multi-tab are one code path.
4. Consistency model before technology (server-authoritative / CRDT / OT); retrofitting one into "just broadcast JSON patches" is a rewrite.
5. Fan-out is a pub/sub layer, not a for-loop; per-connection state on one node is the scaling dead end.

## Transport selection (compressed — cold answers reproduce the reasoning)

Server→client only → **SSE** (protocol-level auto-reconnect + `Last-Event-ID` resume free; plain HTTP infra; ~6-connection limit moot on HTTP/2). Genuinely bidirectional + latency-sensitive → WebSocket (you own heartbeats/reconnect/resume/backpressure). P2P media or late-is-worthless data → WebRTC (budget signaling + STUN/TURN; ~10–20% of enterprise/mobile needs TURN). Updates rarer than ~30s → polling is honest. Managed infra (Ably/PartyKit/Durable Objects) removes most WebSocket operational objections; serverless pushes toward SSE or managed layers.

**WebTransport — currency correction**: cold answers say "Safari never shipped it." **Safari 26.4 shipped WebTransport; it reached cross-browser Baseline in 2026.** Still not the default — UDP:443 is blocked on many corporate/hotel networks, so you need a WebSocket fallback anyway; adopt only for its specific semantics (unreliable datagrams, many independent streams).

## Lifecycle engineering (sharpened anchors)

- Backoff: full jitter `random(0, min(cap, base·2^attempt))`; reset attempts only after the connection has *proven stable* (~30s), not on open — else a connect-drop loop hammers at base delay.
- Heartbeats: TCP keepalive lies; NAT/ALB/nginx idle timeouts commonly ~60s. App-level ping every 15–30s, dead after 2–3 misses; detect *liveness*, not writability. SSE: comment frames (`: ping\n\n`).
- Resume: monotonic per-stream cursor on every event; bounded replay buffer (e.g. Redis Streams, last N minutes); **the cursor-expired → explicit full-resync path is mandatory on day one** — without it clients silently diverge forever.
- Client hygiene: pause on `visibilityState === 'hidden'`; `navigator.onLine` is a hint that lies both ways; share one connection across tabs (`BroadcastChannel`/SharedWorker leader election).

## State sync (compressed) + landscape currency

- Default: server-authoritative — clients send intents, server assigns order, clients reconcile optimistic echoes by idempotency key (tag local echoes so the server broadcast *replaces* the optimistic entry — the classic double-message-in-chat bug).
- CRDT only when concurrent offline-capable editing must merge automatically. 2026 state: **Yjs** production default (Hocuspocus, editor bindings, Liveblocks); **Automerge** for git-like full-history JSON documents; **Loro** (Rust core, Fugue, compact encodings) the fast newcomer. The two underplayed costs: metadata/tombstone growth (schedule snapshot+compact from day one; load-test with a simulated six-month-old doc) and *merge-without-intent* (convergence ≠ semantic correctness). Fine-grained CRDT-op authorization is genuinely hard — validate at the sync server or accept snapshot-level defense.
- Decision shortcut: if a human would need to see both versions to merge, don't pretend a data structure can — locking/presence or LWW-with-history instead.
- OT: only when adopting a system that embodies it (ShareDB); greenfield collaborative text goes CRDT.
- **Sync engines (currency)**: **Zero (Rocicorp) reached 1.0 in 2026** — cold answers still call it pre-stable. ElectricSQL (Postgres shapes, read-path), PowerSync (Postgres/Mongo/MySQL→SQLite bidirectional), Convex (server-first reactive, not offline-first). Honest tradeoff: they collapse transport+cursor+idempotency+cache into one tool at the cost of coupling your data model to a young ecosystem. Read-mostly realtime → SSE + TanStack Query invalidation is dramatically less machinery.

## Ordering, idempotency, presence, fan-out (checklist — mechanisms cold-recoverable)

Per-scope server-assigned sequences (client timestamps unusable); client idempotency keys convert at-least-once into effective exactly-once *processing* (exactly-once *delivery* is a myth); gap detection → fetch missing range over HTTP. Presence is soft state: TTL-expired (heartbeat ~10–30s), never through the durable log, cursors coalesced to ≤10–20Hz latest-wins on a separate channel (Yjs awareness / managed presence — don't persist cursor positions to the database). Fan-out: connection tier ⟂ broker ⟂ app tier; sticky sessions only for stateful fallback stacks (Socket.IO long-polling), not pure WebSocket; bound per-connection send buffers and *disconnect* slow consumers (better a reconnect than an OOM); coalesce high-frequency broadcasts at 50–100ms ticks; per-room actors (Durable Objects/PartyKit) are the simplest correct topology for collaborative docs. Buy below ~10 engineers or when realtime isn't the product's core; keep envelope + cursor semantics provider-agnostic.

## Pitfalls (one-liners)

Auth token in the WS URL query string (logged everywhere; use cookies/one-time ticket/first-message auth — and handle token expiry mid-connection); SSE dead behind buffering middleware (`X-Accel-Buffering: no`, no compression on the stream route, verify through the *production* proxy chain); EventSource/WebSocket opened in a React effect without cleanup (every-message-twice after navigating; one app-level connection); Socket.IO multi-node without the Redis adapter (works in single-instance staging, drops in prod); publish-after-commit without a transactional outbox (crash between commit and publish silently drops events; outbox + relay or CDC); no message envelope/versioning (`{type, v, seq, payload}`, ignore unknown types, keep old shapes parseable one deploy cycle — connected clients don't refresh on your schedule); assuming `onclose` fires (only missed heartbeats are truth); broadcasting full state per change (deltas + sequence; full state is the resync path).

## Micro-example: resumable SSE, the load-bearing lines

```ts
// server: honor Last-Event-ID; replay from bounded buffer; explicit resync event when cursor expired
const cursor = req.headers.get('last-event-id') ?? url.searchParams.get('cursor');
// each event: `id: ${e.seq}\nevent: order\ndata: ${JSON.stringify(e)}\n\n`; every 20s: ': ping\n\n'
// headers: text/event-stream, cache-control: no-cache, x-accel-buffering: no

// client: EventSource reconnects itself and resends Last-Event-ID
es.addEventListener('order', (ev) => {
  localStorage.lastSeq = ev.lastEventId;
  queryClient.invalidateQueries({ queryKey: ['order', JSON.parse(ev.data).orderId] });
});
es.addEventListener('resync', () => queryClient.invalidateQueries()); // cursor-expired path — day one
```
The UI pattern matters as much as the transport: the SSE handler invalidates the query cache rather than patching the DOM — one rendering path for load/reconnect/update.

## Verification — the five chaos questions

Server restarts mid-stream? Client sleeps 10 minutes? Message delivered twice? Out of order? Resume buffer expired? Each needs a designed answer. Plus: retry-every-mutation-twice test (idempotency), 10k-client herd test (jitter + resume served from buffer, not the primary DB), two-tab test, one hostile network. Consistency model named explicitly in the design. Pre-building Kafka fan-out for 200 users is waste.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 12 baseline (cut/compressed), 1 partial (sharpened), 1 delta (expanded).
- The delta: WebTransport currency — cold answers assert Safari hasn't shipped it (Safari 26.4 did; Baseline 2026). Partial: sync-engine landscape (Zero 1.0 in 2026; cold answers say "pushing toward stable").
- Opus cold nails: SSE-vs-WS reasoning, full-jitter backoff + stability-gated reset, heartbeat numbers, mandatory resync path, Yjs/Automerge/Loro + CRDT costs, idempotency/outbox/adapter/presence, chaos tests — kept only as compressed anchors.
