---
name: realtime-web
description: Load when building realtime or collaborative web features — choosing transports (SSE vs WebSocket vs WebRTC vs polling), reconnection/resume logic, live sync architectures (CRDT/OT/server-authoritative), presence, message ordering and idempotency, fan-out scaling, or offline-first sync engines.
---

# Realtime Web

## Core mental model

1. **Realtime is a distributed-systems problem wearing a UI costume.** The hard parts are ordering, idempotency, reconnection, and conflict resolution — not the socket API. Any design that only works when every message arrives exactly once, in order, on a connection that never drops, is a design that fails in production Wi-Fi.
2. **Downstream and upstream are separate decisions.** Most "realtime" features are 95% server→client (feeds, dashboards, notifications, job progress). Client→server can stay plain HTTP (POST) — which keeps auth, retries, load balancing, and observability boring. Only true bidirectional low-latency needs (collab editing cursors, games, trading) justify a socket both ways.
3. **The connection is a cache of a subscription, not the source of truth.** Model client state as "last known snapshot + resume cursor." Then reconnection is just "resubscribe from cursor," multi-tab is coherent, and a cold page load is the same code path as a reconnect.
4. **Consistency model before technology.** Decide who wins concurrent writes — the server (authoritative), a merge function (CRDT), or transformed intents (OT) — before choosing libraries. Retrofitting a consistency model into a "just broadcast JSON patches" system is a rewrite.
5. **Fan-out is a pub/sub layer, not a for-loop.** The moment you run >1 server instance, "send to everyone in room X" requires a broker (Redis pub/sub, NATS, Kafka, or a managed layer). Design channels/topics as first-class from day one; per-connection state on a single node is the scaling dead end.

## Transport selection — the reasoning chain

Ask in order:

1. **Is it server→client only?** → **SSE (`EventSource`) is the underrated default.** Reasons it wins:
   - Plain HTTP: proxies, load balancers, auth middleware, HTTP/2 multiplexing, and CDNs handle it natively; standard observability applies; trivially testable with `curl`.
   - **Automatic browser reconnection with `Last-Event-ID` resume is built into the protocol** — WebSocket gives you neither.
   - It's how LLM token streaming standardized, so infrastructure support keeps improving.
   - Costs: text-only frames, no upstream channel (use POST — usually a feature, see principle 2), and on HTTP/1.1 a ~6-connections-per-origin limit (moot on HTTP/2+, which any 2026 deployment should be).
2. **Genuinely bidirectional and latency-sensitive** (collab cursors, multiplayer, terminal)? → **WebSocket.** Accept its costs knowingly: you own heartbeats, reconnection, resume, backpressure; some corporate proxies/old LBs still mishandle upgrades; sticky routing questions appear at scale.
3. **Peer-to-peer media or unreliable-delivery data** (video calls, voice, game state where late = worthless)? → **WebRTC** (the only browser path to UDP-like semantics and P2P). Budget for signaling server + STUN/TURN; expect ~10–20% of enterprise/mobile networks to need TURN relay. Never choose WebRTC just for "low latency data to server" — operational cost is an order of magnitude higher.
4. **Updates rarer than ~every 30s, or fetch-on-signal pattern?** → **Polling is honest and cheap.** Long-polling remains the fallback transport of last resort behind hostile middleboxes.
5. **WebTransport** (HTTP/3, multiplexed + optional unreliable streams): reached cross-browser Baseline in 2026 (Safari 26.4 shipped it). Still not the default: UDP:443 is blocked on many corporate/hotel networks, so it requires a WebSocket fallback path anyway — adopt only when you need its specific semantics (unreliable datagrams, many independent streams).

What changes the answer: managed infra (Ably/Pusher/PartyKit/Cloudflare Durable Objects, or Phoenix Channels/Elixir if you own the stack) removes most WebSocket operational objections; serverless-only backends push toward SSE or managed layers because long-lived sockets and lambdas mix badly.

## Connection lifecycle engineering

- **Reconnect with exponential backoff + full jitter**: `delay = random(0, min(cap, base * 2^attempt))` (e.g., base 1s, cap 30s). Without jitter, a server restart makes every client reconnect in synchronized waves — the thundering herd that turns a blip into an outage. Reset attempt count only after a connection has proven stable (e.g., held 30s), not on connect — otherwise a connect-drop loop hammers at base delay.
- **Resume tokens**: every event carries a monotonic cursor (per-stream sequence or log offset); client sends the last cursor on reconnect; server replays from a bounded buffer (e.g., last N minutes in Redis Streams). If the cursor has aged out, the server must say so explicitly and the client falls back to **full resync (snapshot) — this path is mandatory**; systems without it silently diverge forever. SSE gives you the plumbing (`id:` field → `Last-Event-ID` header) for free.
- **Heartbeats, because TCP keepalive lies**: a dead connection can look ESTABLISHED for many minutes; NATs and LBs silently drop idle mappings (commonly ~60s idle timeouts; ALBs and nginx `proxy_read_timeout` default to 60s). Application-level ping/pong every ~15–30s, declare dead after 2–3 missed. Detect *liveness*, not just writability — a WebSocket can accept writes into a void. For SSE, send comment frames (`: ping\n\n`) to keep intermediaries from timing out the response.
- **Client-side hygiene**: pause/downgrade on `document.visibilityState === 'hidden'`; use `navigator.onLine` and the `online` event only as *hints* (they lie both ways — verify with a real request); share one connection across tabs where it matters (`BroadcastChannel` or a `SharedWorker`/leader-election) instead of N tabs × M subscriptions.

## State sync architectures — decision reasoning

Ask: **can the server just win?**

- **Server-authoritative (default — most apps)**: clients send intents (HTTP POST or socket message), server validates/applies/assigns order, broadcasts canonical state or deltas; clients apply optimistic updates and reconcile on the server echo (rebase or rollback). Right for dashboards, chat, notifications, e-commerce, most "live" CRUD apps. Simple to reason about, easy to authorize, one source of truth. This is also the model query-layer tools (TanStack Query + invalidation-on-push, Convex's reactive queries) implement for you.
- **CRDT**: needed when *concurrent offline-capable editing of shared data must merge automatically* — collaborative text/canvas/whiteboards. State as of 2026: **Yjs** is the production default (largest ecosystem: y-websocket/Hocuspocus servers, editor bindings for ProseMirror/TipTap, CodeMirror, Slate; managed options like Liveblocks); **Automerge** offers a JSON-document model with git-like full change history (heavier, better for versioned-document use cases); **Loro** (Rust-core, Fugue algorithm for less text interleaving, notably compact encodings) is the fast newcomer with a younger ecosystem. Costs everywhere: metadata growth over document lifetime, and — the underrated one — *merge without conflict is not merge with intent*: two users concurrently editing the same sentence produces valid-but-garbled prose; CRDTs guarantee convergence, not semantic correctness. Also: authorization of fine-grained CRDT updates is genuinely hard (any client can emit any op — validate at the sync server or accept last-line-of-defense snapshots).
- **OT (operational transformation)**: the Google-Docs-era approach; requires a central server transforming ops. In 2026, choose it only when adopting a system that already embodies it (e.g., ShareDB); greenfield collaborative text goes CRDT because the library ecosystem moved there.
- Decision shortcut: *If a human would need to see both versions to merge them, don't pretend a data structure can* — use locking/presence ("Alice is editing") or server-authoritative last-write-wins with history, and reserve CRDTs for text/canvas where character-level merge is genuinely what users want.

## Presence and ephemeral state

Presence (who's online, cursors, typing, selection) is *soft state* — treat it categorically differently from messages:

- It must expire by TTL, never be persisted as truth, and never go through the durable message log.
- Implementation shape: client heartbeats presence every ~10–30s → store with TTL (Redis `SETEX` or a sorted-set indexed by expiry) → broadcast deltas; a missed TTL sweep marks offline. Don't rely on disconnect events alone — they don't fire for sleeping laptops.
- Throttle high-frequency ephemeral streams: cursor positions at ≤ 10–20Hz, coalescing to latest. Dropping intermediate cursor positions is *correct* — they're superseded, not lost.
- Don't send typing indicators through the same ordered channel as messages: ephemeral state wants "latest wins" delivery, durable messages want "all, in order" — different semantics, different channels.
- Yjs's "awareness" protocol and managed presence (Liveblocks, Ably presence sets) implement exactly this split — use them rather than persisting cursor positions to the database (a real and recurring design review find).

## Ordering and idempotency

- **Per-key ordering, not global**: guarantee order within a scope that matters (per document, per conversation) via a per-scope monotonic sequence assigned by the *server/broker* (client timestamps are unusable for ordering — clock skew). Global total order is expensive and almost never required.
- **Client-generated idempotency keys on writes** (UUID per intent, stored server-side with the result): reconnect-and-retry then means duplicates are detected, not double-applied. This single pattern converts "at-least-once delivery" (the only delivery you realistically get) into effective exactly-once *processing*. "Exactly-once delivery" as a transport promise is a myth — design consumers to be idempotent instead.
- **Gap detection**: consumers track last-seen sequence per scope; on a gap, fetch the missing range over HTTP rather than trusting the stream. On out-of-order UI application, either buffer-and-reorder within a small window or design updates to be commutative (state patches with versions, not "append" commands).
- Optimistic UI reconciliation: tag local echoes with the idempotency key so the server broadcast replaces the optimistic entry instead of duplicating it (the classic double-message-in-chat bug).

## Scaling fan-out

- Architecture: **connection tier** (dumb, holds sockets, authenticates, subscribes) ⟂ **pub/sub broker** (Redis pub/sub or Streams, NATS, Kafka for durable logs) ⟂ **application tier** (publishes events). Any node can serve any client; "which node holds Alice's socket" stops mattering for delivery.
- Sticky sessions are needed only for *stateful* transports/fallback stacks (Socket.IO with HTTP long-polling fallback requires them; pure WebSocket on a connection tier doesn't, beyond the connection's own lifetime). Prefer designs where losing a node only forces reconnection (cheap, jittered) rather than state loss.
- Watch for: hot rooms (one topic with 100k subscribers → shard the room or use a broker with fan-out offload); slow consumers exerting backpressure (bound per-connection send buffers and *disconnect* readers that can't keep up — better a reconnect than an OOM); broadcast amplification (N messages × M subscribers; coalesce/batch server-side at e.g. 50–100ms ticks for high-frequency streams).
- Managed shortcut: per-room actor models (Cloudflare Durable Objects / PartyKit) give you a single-threaded authority per document — the simplest correct topology for collaborative docs, since ordering within the room is free.
- Build vs buy: managed realtime (Ably, Pusher, Liveblocks, Supabase Realtime) is usually right below ~10 engineers or when realtime isn't the product's core; the connection tier + broker + resume machinery is undifferentiated heavy lifting. Self-host when message volume makes per-message pricing dominate, data can't transit a third party, or you need custom in-band logic the provider can't run. Either way, keep your message envelope and cursor semantics provider-agnostic so migration stays possible.

## Failure modes & pitfalls

- **Auth token in the WebSocket URL query string** — logged by proxies, LBs, and server access logs. Browsers can't set WS headers, so use cookies, a short-lived one-time ticket fetched over HTTPS and passed in the URL, or authenticate in the first message. And handle *expiry mid-connection*: hours-old sockets outlive tokens; either re-auth in-band on a timer or force reconnect at expiry.
- **SSE dying behind buffering middleware**: nginx `proxy_buffering`, compression layers, and some CDNs buffer the response so events arrive in bursts or never. Fixes: `X-Accel-Buffering: no`, disable compression for the stream route (or use a compression setup that flushes per event), and confirm streaming end-to-end through the *production* proxy chain, not localhost.
- **Leaked connections on unmount/HMR**: an `EventSource`/`WebSocket` opened in a React effect without cleanup duplicates subscriptions on every remount — the "why do I get every message twice after navigating" bug. Return `es.close()` from the effect; keep one app-level connection in a module/store, not per component.
- **Socket.IO multi-node without an adapter**: `io.to(room).emit(...)` only reaches sockets on the local node; you must wire `@socket.io/redis-adapter` (or equivalent) for cross-node rooms. Works perfectly in single-instance staging, drops messages in production — the classic.
- **No backpressure handling**: on the client, check `ws.bufferedAmount` before high-frequency sends; on the server, bound per-connection outbound queues and kill slow consumers. Unbounded queues turn one stalled phone connection into node-wide memory growth.
- **Publishing from the request handler after DB commit** (no transactional outbox): a crash between commit and publish silently drops the event; readers diverge until the next full refresh. Outbox table + relay, or CDC (Debezium-style), when events must track the database.
- **No message schema/versioning**: a deploy changes a payload shape and every connected client throws. Envelope every message (`{type, v, seq, payload}`), ignore unknown types, and keep old shapes parseable for one deploy cycle — connected clients don't refresh on your release schedule.
- **Assuming `onclose` fires**: half-open connections can persist through NAT reboots and sleep/wake; only missed heartbeats are truth (see lifecycle section).
- **CRDT documents growing forever**: op history and tombstones accumulate; without compaction (Yjs update merging / snapshotting, Automerge compressed saves) load times degrade over months. Schedule snapshot+compact from day one, and load-test with a six-month-old simulated doc.
- **Serverless + raw WebSocket mismatch**: lambdas can't hold sockets; you need the platform's managed socket layer (API Gateway WebSockets, Durable Objects) or an external provider — or just use SSE from an edge runtime that supports streaming responses.
- **Broadcasting full state on every change**: works until documents grow; send deltas with sequence numbers, keep full state for the resync path only.

## Offline-first and sync engines (landscape as of 2026)

If requirements include offline writes + multi-device sync + optimistic UI, consider a **sync engine** before hand-rolling the above: **Zero** (Rocicorp; reached 1.0 in 2026 — client-defined synced queries over your Postgres, IndexedDB cache, queued offline writes), **ElectricSQL** (Postgres→client partial replication via declarative "shapes"; read-path sync, writes through your API), **PowerSync** (Postgres/MongoDB/MySQL→SQLite bidirectional, developer-controlled write path), **Convex** (server-first reactive queries — not offline-first, but solves live-updating apps with the least architecture), plus CRDT-native stacks (Yjs/Automerge + persistence) for document apps. The honest tradeoff: sync engines collapse transport+cursor+idempotency+cache into one tool, at the cost of coupling your data model to the engine and living with a young ecosystem. For read-mostly realtime, SSE + TanStack Query invalidation is dramatically less machinery.

## How an expert thinks through this

*Scenario: "Add live order-status updates to our dashboard; later we want collaborative order notes."*

Two features, two consistency models — resist unifying prematurely. Order status: server→client only, updates every few seconds at most, writes stay REST. WebSocket considered — rejected: no upstream need, and we'd inherit heartbeat/reconnect/LB work for nothing. Polling considered — 5s polling × 2k dashboards is fine for the servers but latency-visible and wasteful; rejected mostly because SSE is *equally simple* here. Choose SSE: one `/events?cursor=` endpoint, events carry `id:` = the outbox sequence, nginx `proxy_read_timeout` raised + comment-pings every 20s, browser auto-reconnect sends `Last-Event-ID`, server replays from a Redis Stream keyed per org (bounded 15 min; older → client refetches the dashboard query — full-resync path written on day one). Publishing: order service already writes to Postgres — add a transactional outbox table → a relay publishes to Redis; rejected "publish from the request handler after commit" because a crash between commit and publish silently drops events. UI: SSE handler doesn't patch the DOM; it invalidates the TanStack Query cache for that order — one rendering path for load/reconnect/update. Collaborative notes later: concurrent multi-user text → Yjs + Hocuspocus (or a Durable-Object room) on a *separate* WebSocket, because character merge is a CRDT problem and status events are not; presence via Yjs awareness. Stopping rule: chaos-test — kill the server mid-stream, sleep the laptop 10 min, run two tabs; if state converges within seconds after each and no duplicate orders appear (idempotency keys verified), the lifecycle work is done. Load-testing 100k concurrent sockets is waste until dashboards approach that.

## Worked micro-example

Resumable SSE with typed events (server: any HTTP framework; shown as fetch-handler pseudocode):

```ts
// server
async function events(req: Request): Promise<Response> {
  const cursor = req.headers.get('last-event-id') ?? new URL(req.url).searchParams.get('cursor');
  const stream = new ReadableStream({
    async start(c) {
      const enc = new TextEncoder();
      c.enqueue(enc.encode('retry: 3000\n\n'));
      for (const e of await replaySince(cursor))          // bounded buffer; may throw CursorExpired
        c.enqueue(enc.encode(`id: ${e.seq}\nevent: order\ndata: ${JSON.stringify(e)}\n\n`));
      subscribe(orgId(req), e =>
        c.enqueue(enc.encode(`id: ${e.seq}\nevent: order\ndata: ${JSON.stringify(e)}\n\n`)));
      setInterval(() => c.enqueue(enc.encode(': ping\n\n')), 20_000);
    },
  });
  return new Response(stream, { headers: {
    'content-type': 'text/event-stream', 'cache-control': 'no-cache', 'x-accel-buffering': 'no',
  }});
}

// client — EventSource reconnects itself and resends Last-Event-ID
const es = new EventSource(`/events?cursor=${localStorage.lastSeq ?? ''}`);
es.addEventListener('order', (ev) => {
  localStorage.lastSeq = ev.lastEventId;
  queryClient.invalidateQueries({ queryKey: ['order', JSON.parse(ev.data).orderId] });
});
es.addEventListener('resync', () => queryClient.invalidateQueries()); // cursor-expired path
```

Backoff with full jitter + stability-gated attempt reset (for the WebSocket cases):

```ts
class ReconnectingWS {
  private attempt = 0;
  private stableTimer?: ReturnType<typeof setTimeout>;
  constructor(private url: string, private onMsg: (m: MessageEvent) => void) { this.open(); }
  private delay() { return Math.random() * Math.min(30_000, 1_000 * 2 ** this.attempt); }
  private open() {
    const ws = new WebSocket(this.url);
    ws.onopen = () => {
      // don't reset attempt yet — only after the connection proves stable
      this.stableTimer = setTimeout(() => { this.attempt = 0; }, 30_000);
    };
    ws.onmessage = this.onMsg;
    ws.onclose = () => {
      clearTimeout(this.stableTimer);
      this.attempt++;
      setTimeout(() => this.open(), this.delay());
    };
    // heartbeat: expect a server ping every 20s; declare dead after 2 misses
    let lastSeen = Date.now();
    ws.addEventListener('message', () => { lastSeen = Date.now(); });
    const liveness = setInterval(() => {
      if (Date.now() - lastSeen > 45_000) { clearInterval(liveness); ws.close(); }
    }, 5_000);
  }
}
```

Message envelope that survives deploys and retries:

```json
{ "type": "order.updated", "v": 1, "seq": 40213, "idempotencyKey": "b9c4…", "payload": { "orderId": "…", "status": "shipped" } }
```

## Verification / self-check

Walk the design through these before presenting:
1. **The five chaos questions**: server restarts mid-stream? client sleeps 10 minutes? message delivered twice? messages arrive out of order? resume buffer expired? Each needs a designed answer, not "shouldn't happen."
2. Duplicate-write test: does retrying every mutation twice change state? (If yes: idempotency keys missing.)
3. Herd test: 10k clients reconnecting after a deploy — is there jitter, and can the resume path serve them from cache/buffer rather than the primary DB?
4. Two-tab test and one real hostile network (corporate proxy or phone hotspot) before claiming transport works.
5. Consistency model named explicitly in the design (authoritative / CRDT / OT) with the concurrent-edit story written down.
Stopping rule: the feature survives the chaos questions and a two-client concurrent session without divergence or duplicates. Beyond that, scale work should wait for measured connection counts — pre-building Kafka fan-out for 200 users is waste.
