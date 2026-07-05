---
name: networking-fundamentals
description: Load when debugging network-ish production symptoms (intermittent 502s, connection resets, timeouts, TLS/cert errors, DNS weirdness, slow-only-sometimes) or designing anything with load balancers, proxies, CDNs, keepalives, or DNS/TLS/HTTP version choices.
---

# Networking Fundamentals

## Core mental model

- **Every request is a pipeline of independently failing layers: DNS → TCP (or QUIC) → TLS → HTTP → application.** Debugging means localizing which layer failed, and each layer has a dedicated interrogation tool: `dig` for DNS, `nc -vz`/`mtr` for reachability and path, `openssl s_client` for TLS, `curl -v` (which walks the whole pipeline and narrates it) for the composite. The most common debugging failure is interrogating the wrong layer — staring at app logs for what is a DNS TTL problem.
- **"The network" is usually a middlebox.** Between client and server sit load balancers, reverse proxies, NAT gateways, corporate firewalls, service meshes, CDNs — each with its own timeouts, buffer limits, and connection tables. Mysterious intermittent failures are overwhelmingly *disagreements between adjacent boxes* (idle timeout mismatches, size limits, protocol downgrades), not packet loss. Your first move on any intermittent failure: draw every hop between client and origin, and write each hop's idle timeout next to it.
- **Connections are cached state, and cached state goes stale.** Keepalive pools, DNS caches, TLS session tickets, NAT table entries — all performance optimizations that create a second failure mode: the cached thing silently died or moved, and the *next* use fails. "Works on retry" is the signature of stale connection state, not flaky networks.
- **Failures at layer N often present as symptoms at layer N+1.** A TCP RST presents as an HTTP client exception; a DNS change presents as "some pods hit the old backend"; an MTU black hole presents as "small responses work, large ones hang." When a symptom makes no sense at its own layer, go down one.
- **Latency is round trips, not bandwidth.** A fresh HTTPS request costs DNS (0–1 RTT) + TCP (1 RTT) + TLS 1.3 (1 RTT) + HTTP (1 RTT) before any byte of body. At 80ms RTT that's ~240–320ms of pure protocol before the server even thinks. This is why connection reuse, TLS 1.3, and CDNs (which move the RTT-bearing endpoint close to the user) dominate real-world latency work — and why "add more bandwidth" fixes almost nothing interactive.

## The request's journey, with the tool for each stage

1. **DNS** — `dig +short api.example.com`, `dig api.example.com @1.1.1.1` (bypass local cache to compare), `dig +trace` (walk delegation from the root when you suspect the authoritative chain), `dig CNAME`, `dig -x IP` (reverse). Check: does the name resolve, to what, from *where* (resolvers differ), and what TTL is being served?
2. **Reachability / path** — `nc -vz host 443` (can I open a TCP connection at all — separates "route/firewall problem" from everything above it), `mtr host` (live per-hop loss/latency; read it correctly: loss at an intermediate hop that *disappears* at later hops is just routers deprioritizing ICMP to themselves — only loss that *persists to the final hop* is real).
3. **TLS** — `openssl s_client -connect host:443 -servername host` (the `-servername` sets SNI; forgetting it is the classic way to get the wrong cert back and misdiagnose). Read: the cert chain presented, `Verify return code`, negotiated protocol/cipher. Add `-showcerts` to dump the chain, pipe a cert through `openssl x509 -noout -dates -ext subjectAltName` for expiry and SAN checks.
4. **HTTP** — `curl -v https://host/path` narrates all four stages: resolution, connect, handshake (cert subject, ALPN result), request/response headers. `curl --resolve host:443:1.2.3.4` pins DNS so you can test a specific backend through the real TLS/HTTP path (indispensable for testing before a DNS cutover). `curl -w '%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer}\n' -o /dev/null -s` gives you a per-stage latency breakdown — this one-liner localizes "slowness" to DNS vs connect vs TLS vs server-think-time in a single run.

Localization heuristic: name doesn't resolve → DNS. Resolves but `nc` fails → routing/firewall/security group. TCP opens but TLS fails → certs, SNI, protocol mismatch, or a middlebox doing TLS interception. TLS fine but HTTP errors → now, finally, it's the application (or the proxy in front of it).

## DNS reasoning

- Records applied: A/AAAA (name→IP), CNAME (alias — cannot coexist with other data at the same name, which is why apex domains can't CNAME and cloud providers invented ALIAS/ANAME flattening), NS (delegation), MX, TXT (SPF/DKIM/verification), SRV, CAA (which CAs may issue for you — set it).
- **TTL strategy is a rollback-speed dial.** TTL is the maximum staleness of the world's view of your name. Steady state: 300–3600s. Planned cutover: drop TTL to 60s *at least one old-TTL period before* the change (a TTL drop propagates only as fast as the old TTL — this trips everyone), migrate, verify, raise it back. Assume a long tail of resolvers that clamp or ignore low TTLs, so never design a system where correctness (rather than convenience) depends on fast DNS propagation — keep the old backend serving during the tail.
- **Negative caching trap:** NXDOMAIN answers are cached too, with the TTL taken from the zone's SOA. Query a name *before* creating it (typo test, premature deploy) and resolvers can remember it doesn't exist for the whole negative TTL — "I created the record, it still doesn't resolve" is usually this. Check with `dig` and look at the SOA in the authority section of the NXDOMAIN reply.
- **Split-horizon:** internal and external resolvers answer the same name differently (corp VPN gives private IPs). Symptom signature: "works on VPN, fails off" or a pod resolving a peer to a public IP and paying the NAT/firewall toll. Always ask `dig` *from where the failing client runs* — resolution is a function of the resolver, not the name. In Kubernetes, remember `ndots:5` in the default resolv.conf makes short names spray search-domain lookups before trying the literal name — a source of both latency and confusing NXDOMAIN logs; use FQDNs with a trailing dot in hot paths.

## TLS in practice

- TLS 1.3 is the default of the modern web; TLS 1.2 is the compatibility floor; 1.0/1.1 are formally deprecated (RFC 8996). 1.3 cut the handshake to 1 RTT and removed renegotiation and the weak-cipher zoo — if you see a 1.2-only endpoint in 2026, it's a legacy appliance or an interception middlebox, and both are worth flagging.
- As of 2026, post-quantum *hybrid* key exchange (X25519MLKEM768) is live at scale: Chrome (since 124) and Firefox (since 132) offer it by default, OpenSSL 3.5+ supports it, and a meaningful fraction of TLS 1.3 handshakes negotiate it. Practical consequences: ClientHellos got bigger (can exceed one packet — has exposed latent middlebox bugs that assume single-packet Hellos), and "harvest-now-decrypt-later" is the threat it addresses. You mostly get it for free by keeping your TLS stack current.
- **SNI** is how one IP serves many certs: the client names the host in the ClientHello, the server picks the cert. No/wrong SNI → default cert → mysterious mismatch errors that only occur from non-browser clients (old SDKs, health checkers, `openssl s_client` without `-servername`).
- **Chain validation failures** are the #1 TLS incident: the server must send leaf + intermediates (not the root). Browsers self-heal via AIA-chasing and cached intermediates; strict clients (Java, Go, Python, curl) don't — signature symptom: "works in Chrome, fails from the backend/CLI with `unable to get local issuer certificate`." Diagnose with `openssl s_client -showcerts`; fix by serving the full chain (`fullchain.pem`, not `cert.pem` — the exact Let's Encrypt/certbot mistake).
- **mTLS** (both sides present certs) is for service-to-service identity, not browsers. The reasoning: use it where you control both endpoints and want identity without shared secrets — typically via a mesh (Istio/Linkerd) that automates the actual hard part, which is not the handshake but issuance, rotation, and revocation of millions of short-lived certs. Hand-rolled mTLS with year-long certs and no rotation story is a scheduled outage.
- Certificate lifetimes keep shrinking (the CA/Browser Forum has adopted a schedule stepping public-cert maximum validity down toward ~47 days by 2029, as of 2026). Design consequence: cert renewal must be fully automated (ACME) and *alerted on* — expiry-based outages are entirely self-inflicted and still endemic. Monitor `not_after` on every public endpoint, alert at 2 weeks.

## HTTP versions applied

- **1.1**: one request at a time per TCP connection (pipelining is dead) → clients open ~6 parallel connections per host. Head-of-line blocking at the *application* level: a slow response stalls the connection.
- **2**: many streams multiplexed on one TCP connection, header compression (HPACK). Kills app-level HOL blocking — but inherits *TCP-level* HOL: one lost packet stalls every stream on the connection, so h2 degrades badly on lossy links. Server-push is dead (Chrome removed it); prioritization is unevenly implemented.
- **3 / QUIC**: streams over UDP with per-stream loss recovery (no transport HOL), 1-RTT (or 0-RTT resumed) combined transport+TLS handshake, and connection IDs that survive IP changes (mobile Wi-Fi↔cellular migration). TLS 1.3 is built in.
- **Adoption reality as of mid-2026:** HTTP/3 usage has *plateaued* rather than conquered — measurement-dependent figures put it around 21–35% of traffic (Cloudflare's edge share actually peaked around 2023 and slightly declined), with support concentrated at CDNs and hyperscalers. Two reasons worth knowing: UDP is still deprioritized/blocked by some networks and appliances, and QUIC's userspace processing costs it real throughput on fast, clean links — on high-bandwidth low-loss paths (≳500 Mbps fiber) h2/TCP with kernel offloads can outperform h3, flipping QUIC's advantage. Decision rule: h3's wins are *lossy and high-RTT last miles* (mobile, satellite, bad Wi-Fi) and *fresh-connection latency*; enable it at your CDN edge (a checkbox — browsers race h3/h2 via Alt-Svc and fall back cleanly, so it's nearly free) and don't bother inside the datacenter, where gRPC-over-h2 remains the norm.

## Load balancing

- **L4 (connection-level: NLB, HAProxy TCP mode, IPVS)** routes packets/connections without reading them: cheap, protocol-agnostic, preserves end-to-end TLS. **L7 (request-level: ALB, Envoy, nginx)** terminates HTTP(S), so it can route per-path/header, retry, and emit request metrics — at the cost of a TLS hop, config surface, and being itself a thing that 502s. Choose by asking "do I need to see requests to make the routing/observability decision?" — if not, L4 is less machinery to break. Common composite: L4 in front, L7 fleet behind it.
- **Critical L7 subtlety:** the LB multiplexes many client connections onto pooled *backend* connections; client-connection count and backend-connection count are unrelated, and each leg has its own keepalive/idle settings — the root of the 502 pathologies below.
- **Algorithms:** round robin assumes uniform request cost (false for APIs with mixed cheap/expensive endpoints); **least-outstanding-requests** self-corrects for slow backends and is the right default when available; consistent hashing when you need affinity to warm caches/shards (accept the hot-key risk); avoid session stickiness as a *correctness* mechanism — externalize session state instead, keep stickiness as a cache optimization only.
- **The health-check design problem:** a shallow check (`/healthz` returns 200) detects dead processes only; a deep check (verify DB, cache, dependencies) detects real unreadiness but creates the classic own-goal — the *database* blips, every instance's deep check fails simultaneously, the LB removes *all* backends, and a partial degradation becomes a total outage. Rules: liveness = shallow (is the process functional); readiness may be deeper but must **never include shared dependencies** whose failure would fail every instance at once (fail-open or degrade instead); health endpoints must be trivial-cost (they're called by every LB node × frequency); and check the LB's *panic/fail-open* behavior (many, e.g., Envoy's panic threshold, route to all hosts when too many look unhealthy — know whether yours does).
- **Connection draining:** removal from rotation must mean "stop *new* work, finish in-flight" with a deregistration delay ≥ your longest legitimate request; pair with the pod's `preStop` sleep + `terminationGracePeriodSeconds` in Kubernetes so the endpoint is out of the LB *before* SIGTERM lands — the missing `preStop` is why "we get a burst of 502s on every deploy."

### Long-lived streams and client-IP realities

- WebSockets/gRPC streams/SSE through L7 infra hit a different timeout class: per-*request* idle timeouts apply to the whole stream (an ALB's 60s idle timeout kills a quiet WebSocket; so does nginx's `proxy_read_timeout`). Either send protocol-level pings under the smallest per-request timeout, or raise stream-route timeouts specifically — not globally, or you've disabled slow-request protection everywhere.
- Long-lived connections also *defeat rebalancing*: a 24-hour gRPC channel pins its load to whichever backend it landed on at connect time, so scale-out doesn't help existing hotspots. Fixes: server-side `MAX_CONNECTION_AGE` (gRPC) / periodic GOAWAY so clients reconnect and re-balance, or an L7 proxy that balances per-*request* across h2 connections (Envoy does; plain L4 in front of gRPC does not).
- Behind any proxy/LB, the TCP peer address is the proxy, not the user. Client IP arrives via `X-Forwarded-For` (L7) or PROXY protocol (L4). Two standing bugs: rate-limiting/allowlisting on the connection IP (you're limiting your own LB — one noisy user triggers a limit that blocks everyone), and trusting XFF blindly (clients can send a fake first entry; take the *rightmost* entry added by *your* trusted tier, and configure the trusted-proxy count explicitly).

## Proxies, NAT, and the keepalive arithmetic

Every middlebox holding connection state has an idle timeout, and **an idle connection is killed by the hop with the smallest timeout — sometimes silently** (NAT gateways and some firewalls drop the mapping without sending RST/FIN). The client's pool still believes the connection is good; the next request sails into a black hole and waits out its full timeout, or dies with `Connection reset by peer` if something does send RST.

The arithmetic that prevents the whole class:
1. Enumerate hops: client pool → LB → (mesh sidecar) → server; note each idle timeout. Reference points (as of 2026): AWS ALB idle default 60s; AWS NAT Gateway 350s fixed for TCP; Azure LB default 4 min; nginx `keepalive_timeout` default 75s upstream-side `keepalive_timeout` 60s; Node.js server `keepAliveTimeout` default 5s (a famous 502 factory behind 60s LBs).
2. Enforce the invariant **server-side timeout > LB timeout > client-side pool idle timeout**, each hop pair with comfortable margin. The *upstream* member of each pair must outlive the downstream one, so the side that initiates reuse never reuses a connection its peer already closed.
3. For long-idle links you can't tune end-to-end (DB connections through NAT, message-bus consumers), send keepalives *below* the smallest middlebox timeout: TCP keepalive (`tcp_keepalive_time` — Linux default 7200s is uselessly long; set ~60–300s via socket options) or protocol-level pings (gRPC/h2 PING, websocket ping).

## TCP realities that surface in production

- **RST vs FIN tells you intent.** FIN = orderly close ("I'm done sending"); RST = abort ("no such connection / go away now"). A client seeing `Connection reset by peer` means something *actively* refused state it was expected to have: a process crashed mid-connection, a backend closed with unread data in the buffer (common when servers close on request-size violations), or a middlebox with no matching table entry answered a packet on a mapping it already forgot. `ECONNREFUSED` (RST to your SYN) is different and simpler: nothing is listening on that port — wrong port, crashed process, or wrong host.
- **TIME_WAIT and ephemeral port exhaustion:** the side that closes first holds the socket in TIME_WAIT (~60s Linux). A proxy/client machine opening thousands of short-lived outbound connections to one destination burns through the ~28k default ephemeral ports (`net.ipv4.ip_local_port_range`) and new connects fail with `EADDRNOTAVAIL` — under load only, mysteriously. Fix is connection reuse (keepalive pools), not sysctl heroics; `net.ipv4.tcp_tw_reuse=1` helps for outbound, but a service doing 500 connects/sec to one upstream *needs a pool*, full stop. Count: 28,000 ports / 60s TIME_WAIT ≈ 466 new connections/sec ceiling per (src IP, dst IP, dst port) tuple.
- **Conntrack is a hidden capacity limit:** any Linux box doing NAT or stateful firewalling (including every Kubernetes node running kube-proxy) tracks each flow in a fixed-size table (`nf_conntrack_max`). Full table = new connections silently dropped, `nf_conntrack: table full, dropping packet` in dmesg — the symptom is "random timeouts under load" and nobody thinks to look in dmesg. Check `conntrack -C` against the max during load tests.
- **SYN retries explain weird timeout durations.** Unanswered SYNs retry on an exponential schedule (~1s, 2s, 4s...); Linux default 6 retries ≈ 127s to `ETIMEDOUT`. When your "10s connect timeout" mysteriously takes 127s, no one set a connect timeout at all — the OS default is what you're seeing. Always set explicit connect timeouts (1–3s intra-DC) separate from request timeouts.

## CDN mental model

- A CDN is a distributed cache plus a TLS/TCP terminator near the user. Even for *uncacheable* APIs it pays: the user's TCP+TLS round trips happen over a short RTT, and edge→origin rides warm, pooled connections.
- **The cache key is the contract.** Default ≈ host + path (+ query, configurable). Everything not in the key is *invisible to caching* — vary by header/cookie without declaring it and users receive each other's responses (the classic leak: caching `Cache-Control`-less authenticated responses). Everything *needlessly* in the key (irrelevant query params, `utm_*`) shreds hit rate. Explicitly decide, per route: what's in the key, what's normalized out, and what `Vary` declares.
- **Invalidation:** purge-by-URL is fast but you must know every URL variant; purge-by-tag/surrogate-key (Fastly, Cloudflare) is the scalable pattern — tag responses with entity IDs, purge the tag on write. Better default for APIs: short TTL + `stale-while-revalidate` (serve stale, refresh in background) — bounded staleness with zero invalidation machinery. Reserve purging for correctness-critical changes; treat "we'll purge on every write" designs as suspect (purge latency and fan-out become your consistency model).
- **Origin shield / tiered caching:** a designated mid-tier POP that collapses many edge misses into one origin fetch. Turn it on when origin load scales with POP count or when a cold/purged object triggers a request stampede (request coalescing at the shield absorbs it).

## How an expert thinks through it: "we get intermittent 502s, ~0.2%, no pattern"

Intermittent + low-rate + LB in front: prior #1 is a connection-reuse race, not the app. First, evidence over vibes: the ALB's own logs classify each 502 — check the target-connection error field, and correlate 502 timestamps with target deploys/scale-in events. Three hypotheses, ordered by base rate: (a) keepalive mismatch — backend closes idle connections sooner than the ALB expects, ALB reuses a corpse → immediate 502; (b) deploy/scale-in without draining — 502 bursts clustered at rollout times; (c) actual app crashes — but that would show in app error logs, and it doesn't. Check (a) cheaply: backend is Node — `server.keepAliveTimeout` unset → 5s, ALB idle 60s. Invariant violated (server must outlive LB). That alone explains a steady trickle. Fix: `keepAliveTimeout = 65_000`, `headersTimeout = 66_000`. Rejected along the way: "add retries at the ALB" — masks it, and retrying non-idempotent POSTs is a correctness bug; "it's AZ packet loss" — rejected because loss would show as latency/timeouts too, and `mtr` between AZs is clean. Deploy the timeout fix, watch the 502 rate: trickle gone, but small bursts remain exactly at deploy times → that's hypothesis (b): add the `preStop` sleep so pods leave the target group before SIGTERM. Both fixed, rate is 0.00x% — remaining singletons correlate with target OOM restarts, which is a different (capacity) ticket. Stop here: don't chase asymptotic zero through retry layers that hide real failures.

## Production mysteries: signature → likely cause → first check

| Signature | Prior | First check |
|---|---|---|
| Intermittent 502s, steady trickle | Keepalive timeout mismatch (backend closes before LB) | Compare backend idle timeout vs LB idle timeout; LB logs' error-cause field |
| 502s bursting at deploy times | No draining/preStop; pods die with in-flight requests | Correlate 502 timestamps with rollout events; check deregistration delay & preStop |
| `Connection reset by peer` after a *consistent* idle interval | NAT/firewall reaped the idle mapping | Measure the interval; compare to NAT/firewall timeouts; add keepalives below it |
| Works on retry, fails first try after quiet periods | Stale pooled connections (same family as above) | Pool idle-eviction setting vs path's smallest middlebox timeout |
| Small responses fine, large ones hang | PMTUD black hole (filtered ICMP, tunnel/VPN in path) | `ping -M do -s 1472`, bisect size; clamp MSS on the tunnel |
| Slow only from some networks/regions | Path or last-mile issue; or h3/UDP blocked forcing fallback | RUM breakdown by ASN/geo; `mtr` from an affected vantage |
| Slow first request, fast after | Cold: DNS miss + TCP + TLS + cold upstream pool | `curl -w` per-stage timing, fresh vs warm |
| p99 slow, p50 fine, CPU idle | Queueing somewhere: listener backlog, pool contention, one slow backend in rotation | Per-backend latency split at the LB; pool wait-time metrics |
| "Random" timeouts under load, dmesg mentions conntrack | Conntrack table full on a NAT/k8s node | `conntrack -C` vs `nf_conntrack_max` during load |
| New connects fail under load from one busy client (`EADDRNOTAVAIL`) | Ephemeral port exhaustion / TIME_WAIT pileup | `ss -s` state counts; introduce/repair connection pooling |
| Exactly ~127s hangs | No connect timeout set; OS SYN-retry default | Set explicit connect timeouts everywhere |

Use the table as priors to order checks, not as a verdict — confirm with the layer tools before fixing.

## Failure modes & pitfalls

- **Testing TLS without SNI** (`openssl s_client` minus `-servername`, monitoring probes hitting the IP) → get the default cert → false "wrong certificate" alarm, or worse, a probe that passes while real clients fail.
- **Serving leaf-only cert chains.** Works in browsers (AIA chasing/cache), fails in Go/Java/Python clients. Always deploy the full chain; verify with `openssl s_client -showcerts` from a *strict* client's viewpoint.
- **Dropping DNS TTL at cutover time.** The old 24h TTL is already in the world's caches; your 60s TTL applies only after they expire. Lower TTL ≥ one old-TTL ahead; keep the old target alive through the tail.
- **Trusting mid-path `mtr` loss.** 30% "loss" at hop 6 that vanishes by hop 12 is ICMP deprioritization, not packet loss. Only final-hop-persistent loss indicts the path.
- **Deep health checks including shared dependencies** → correlated removal of every backend when the dependency blips → self-inflicted total outage. Readiness checks own-instance resources only.
- **`Connection reset by peer` after exactly N minutes idle** → a middlebox (NAT gateway ≈ 350s on AWS, firewall, LB) reaped the mapping. Fix with keepalives under the smallest timeout, not with retries alone. The tell is the *consistency* of the idle interval.
- **Large payloads hang, small ones fine** → MTU/PMTUD black hole (ICMP "frag needed" filtered somewhere, common with VPNs/overlays). Test: `ping -M do -s 1472 host` and bisect; fix with clamped MSS on the tunnel, not by "raising timeouts."
- **Retries at three layers** (client SDK, mesh sidecar, LB) multiply: one slow backend event becomes 3×3×3 = 27 requests — a self-inflicted retry storm. Budget retries at *one* layer (usually the client or the mesh, not both), always with jittered backoff, never on non-idempotent methods without an idempotency key.
- **Pool sized above the server's limit:** 50 app pods × 20 pooled DB connections = 1000 > Postgres `max_connections` 500 — works until a deploy doubles live pods briefly. Do the multiplication; it's the whole diagnosis.
- **Assuming h3 = faster everywhere.** On fast clean links QUIC's userspace cost can make it *slower* than h2 (as of 2026 this is a measured, mainstream result). Enable h3 at the edge for lossy/mobile win, keep h2 inside the DC, and A/B it with RUM data rather than believing either camp.
- **Caching authenticated responses at the CDN** because nobody set `Cache-Control: private/no-store` and the cache key ignores the cookie. Audit every cacheable route for "what distinguishes users, and is it in the key or in `Vary`?"
- **Clients that cache DNS forever.** The JVM historically cached successful lookups indefinitely under a security manager (`networkaddress.cache.ttl`), and any app that resolves once at startup and holds the IP (common in hand-rolled connection code) never sees your failover. Symptom: you failed over via DNS, most clients moved, one fleet didn't until restarted. Fix at the client (respect TTLs, re-resolve on connection error) — this is exactly why DNS-based failover needs a connection-level fallback story.
- **IPv6/Happy Eyeballs half-configured**: AAAA records published but the v6 path is broken (firewall, missing routes). Modern clients race v4/v6 and mostly hide it as "slow sometimes"; strict or older clients hang. If you publish AAAA, monitor the v6 path as a first-class endpoint — or don't publish it.

## Worked micro-examples

**1. Per-stage latency localization with one curl:**

```bash
curl -o /dev/null -s -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://api.example.com/v1/items
# dns=0.912 tcp=0.964 tls=1.083 ttfb=1.104 total=1.412
```

Reading it (values are cumulative): DNS took 912ms — that's the whole problem (healthy is single-digit ms warm, <100ms cold; ~5s means a resolver timed out and a fallback answered). TCP connect added 52ms (≈ the RTT), TLS 119ms (~2×RTT — fine for a fresh 1.3 handshake... suspicious if you expected session resumption), server think-time 21ms (ttfb−tls). Run it 20× in a loop before concluding anything from one sample; bimodal results point at cache expiry or load-balanced backends that differ.

**2. The keepalive arithmetic, applied (Node.js behind an AWS ALB).** Invariant: server idle timeout > ALB idle timeout > client pool idle timeout.

```js
// ALB idle timeout: 60s (default). Node defaults: keepAliveTimeout 5s -> violates invariant, 502 factory.
const server = app.listen(8080);
server.keepAliveTimeout = 65_000;   // > ALB's 60s: ALB, not the server, retires connections
server.headersTimeout   = 66_000;   // must exceed keepAliveTimeout (Node quirk: guards the same race on header reads)
```

And the outbound side of the same app calling an internal service through the mesh (client must be the impatient one): pool idle timeout 30s < upstream Envoy/nginx 60s < server 75s.

**3. Reading a chain failure with s_client:**

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts </dev/null 2>/dev/null | head -20
# Certificate chain
#  0 s:CN = api.example.com          <- leaf only; no "1 s: ... R11/intermediate" line follows
# Verify return code: 21 (unable to verify the first certificate)
```

One cert in the chain + code 21 = server is sending the leaf without intermediates: browsers will paper over it, Go/Java/Python clients will fail. Fix is deploying `fullchain.pem`. Contrast: `Verify return code: 10 (certificate has expired)` — check *which* cert expired (`| openssl x509 -noout -dates` per chain element); an expired *intermediate* with a valid leaf is a CA-bundle/renewal-pipeline issue, not a leaf renewal.

## Verification / self-check

- Localize before you theorize: run the four-stage interrogation (`dig` → `nc`/`mtr` → `s_client` → `curl -v` + `-w` timings) *from the failing vantage point* — resolution, routing, and MTU are all position-dependent, so a check from your laptop only vouches for your laptop.
- For any intermittent-failure diagnosis, demand the two signatures match: the *rate* (does the timeout arithmetic predict roughly this frequency?) and the *timing* (do failures cluster at deploys/idle-intervals/TTL boundaries as the theory requires?). A theory that doesn't predict the observed clustering is the wrong theory, however plausible.
- After a fix, verify at the layer that failed (e.g., 502 rate by cause code in LB logs, not just "app errors down") and hold a soak long enough to cover the failure's natural period (an idle-timeout bug needs quiet hours to re-trigger; a deploy-burst bug needs a deploy).
- Stopping rule: stop when the remaining error rate is explained (each residual failure attributable to a known, accepted cause) — not when it's zero. Chasing the last 0.01% with extra retry layers trades visible, diagnosable failures for hidden latency and masked bugs.
