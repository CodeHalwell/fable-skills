---
name: networking-fundamentals
description: Load when debugging network-ish production symptoms (intermittent 502s, connection resets, timeouts, TLS/cert errors, DNS weirdness, slow-only-sometimes) or designing anything with load balancers, proxies, CDNs, keepalives, or DNS/TLS/HTTP version choices.
---

# Networking Fundamentals

## The standard doctrine, compressed

A strong model already produces this cold; anchors only. Every request is DNS → TCP/QUIC → TLS → HTTP → app; localize the failing layer with `dig` → `nc`/`mtr` → `openssl s_client -servername` → `curl -v`, run *from the failing vantage point* (pod resolver/routes/MTU ≠ your laptop; `kubectl debug`). Intermittent failures are usually middlebox disagreements (idle-timeout mismatches), not packet loss; "works on retry" = stale pooled connection. The classics a strong model nails with exact numbers: ALB 60s vs Node `keepAliveTimeout` 5s 502 factory (fix: 65s/66s, invariant server > LB > client pool); AWS NAT Gateway 350s silent mapping reap (keepalives under 350s); PMTUD black hole through tunnels (`ping -M do -s 1472`, clamp MSS); mid-path `mtr` "loss" that vanishes downstream = ICMP deprioritization; leaf-only chain works-in-Chrome-fails-in-Go (serve `fullchain.pem`; AIA chasing); DNS cutover TTL-drop ≥ one old-TTL ahead + NXDOMAIN negative caching from SOA; deep health checks on shared dependencies → correlated full-fleet removal (readiness checks own-instance resources only; know your LB's panic/fail-open threshold); ephemeral-port math 28k ports ÷ 60s TIME_WAIT ≈ 466 conn/s ceiling (fix: pooling, `tcp_tw_reuse` for outbound); 127s hang = 6 SYN retries, nobody set a connect timeout; `ndots:5` search-domain spray in k8s (FQDN trailing dot or `dnsConfig`); X25519MLKEM768 hybrid PQ live by default and its oversized-ClientHello middlebox breakage; CA/B cert lifetimes stepping to ~47 days by 2029 → ACME automation + expiry alerting mandatory.

The quiver (run these, in this order, before theorizing):

```bash
dig +short api.example.com                      # this host's resolver
dig api.example.com @1.1.1.1 +short             # vs public resolver (split-horizon/cache check)
nc -vz api.example.com 443                      # will TCP even open
mtr -rwbz -c 50 api.example.com                 # per-hop; only final-hop-persistent loss is real
openssl s_client -connect h:443 -servername h -showcerts   # chain + verify code (21=missing intermediate, 10=expired — check WHICH cert)
curl --resolve h:443:10.0.0.5 https://h/path    # pin DNS: test a backend through real TLS pre-cutover
curl -o /dev/null -s -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer}\n' https://h/p
ss -tnp | grep :5432 ; conntrack -C             # socket states; conntrack fill
```

The `curl -w` values are cumulative; run 20×, bimodal results mean cache expiry or divergent backends.

## Corrections and sharpenings

- **HTTP/3 plateaued; don't extrapolate the 2021 hype curve.** The baseline story ("h3 adoption steadily growing") is stale: measurement-dependent share sits ~21–35% of traffic as of mid-2026, and **Cloudflare's edge h3 share actually peaked around 2023 and slightly declined**. Mechanism to internalize: QUIC's userspace processing loses to TCP's kernel offloads on fast clean links (≳500 Mbps fiber), flipping the advantage — h3 wins on lossy/high-RTT last miles and fresh-connection latency only. Decision: enable at the CDN edge (checkbox; Alt-Svc race + clean fallback), keep h2 inside the DC, A/B with RUM rather than believing either camp.
- **Conntrack is the invisible ceiling on every k8s node.** Any box doing NAT/stateful firewalling (= every kube-proxy node) tracks flows in a fixed table (`nf_conntrack_max`); full table = new connections *silently dropped*. Symptom is "random timeouts under load" and the evidence is only in `dmesg` (`nf_conntrack: table full`), where nobody looks. Check `conntrack -C` vs max during load tests.
- **Retries multiply across layers.** Client SDK × mesh sidecar × LB each retrying 3× turns one slow backend event into 27 requests — a self-inflicted storm that also masks the underlying failure. Budget retries at exactly one layer, jittered backoff, never on non-idempotent methods without an idempotency key. (Same discipline for "add ALB retries" as a 502 fix: it hides the keepalive bug and duplicates POSTs.)
- **Do the pool multiplication.** 50 pods × 20 pooled DB connections = 1000 > Postgres `max_connections` 500 — works until a deploy briefly doubles live pods. The whole diagnosis is one multiplication; check it before anything clever.
- **Long-lived streams defeat both timeouts and rebalancing.** Per-request idle timeouts apply to the *whole* stream (ALB 60s kills a quiet WebSocket; nginx `proxy_read_timeout` too) — ping under the smallest timeout or raise it on stream routes only, never globally. And a 24h gRPC channel pins load to wherever it connected: scale-out does nothing for existing hotspots. Fix: server-side `MAX_CONNECTION_AGE`/periodic GOAWAY, or an L7 proxy that balances per-request across h2 (Envoy does; L4 in front of gRPC does not).
- **Client IP behind proxies, the two standing bugs:** rate-limiting on the connection IP limits your own LB (one noisy user blocks everyone), and trusting `X-Forwarded-For` blindly lets clients forge the first entry — take the rightmost entry added by *your* trusted tier with an explicit trusted-proxy count.
- **Clients that cache DNS forever break failover.** JVM historical indefinite caching (`networkaddress.cache.ttl`), and any app that resolves once at startup and holds the IP: you fail over via DNS, one fleet doesn't move until restarted. DNS-based failover needs a connection-level fallback story (re-resolve on connection error).
- **AAAA half-published is a slow-burn outage.** Happy Eyeballs mostly hides a broken v6 path as "slow sometimes"; strict/older clients hang. Publish AAAA only if you monitor the v6 path as a first-class endpoint.

## Signature → prior → first check (order checks by this, then confirm at the layer)

| Signature | Prior | First check |
|---|---|---|
| Intermittent 502s, steady trickle | Backend idle timeout < LB idle timeout | LB log error-cause field; compare the two timeouts |
| 502 bursts at deploy times | No draining/preStop before SIGTERM | Correlate with rollouts; deregistration delay + preStop sleep |
| Reset after a *consistent* idle interval | NAT/firewall reaped mapping (AWS NAT 350s) | Measure the interval; keepalives below it |
| Works on retry after quiet periods | Stale pooled connections | Pool idle-eviction vs smallest middlebox timeout |
| Small OK, large hangs | PMTUD black hole (tunnel/VPN) | `ping -M do -s 1472`, bisect; clamp MSS |
| Slow first request, fast after | Cold DNS+TCP+TLS+pool | `curl -w` fresh vs warm |
| p99 slow, p50 fine, CPU idle | Queueing: backlog, pool contention, one slow backend | Per-backend latency split at LB; pool wait metrics |
| Random timeouts under load | Conntrack full | `conntrack -C` vs max; dmesg |
| `EADDRNOTAVAIL` from one busy client | Ephemeral ports / TIME_WAIT | `ss -s`; add pooling |
| Exactly ~127s hangs | No connect timeout set anywhere | Set explicit connect timeouts (1–3s intra-DC) |
| Slow/NXDOMAIN spam from pods | `ndots:5` search spray | resolv.conf; trailing-dot FQDNs |

## Verification / self-check

- Demand the two signatures match before accepting a diagnosis: the *rate* (does the timeout arithmetic predict this frequency?) and the *timing* (do failures cluster at deploys/idle intervals/TTL boundaries as the theory requires?). A theory that doesn't predict the observed clustering is wrong, however plausible.
- Verify the fix at the layer that failed (502 rate *by cause code* in LB logs, not "app errors down") and soak across the failure's natural period (idle bugs need quiet hours; deploy bursts need a deploy).
- Stopping rule: stop when the residual error rate is *explained* (each failure attributable to a known accepted cause), not zero — chasing the last 0.01% with retry layers trades diagnosable failures for hidden latency and masked bugs.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 12 baseline (cut/compressed — every classic signature incl. exact constants: Node 5s/ALB 60s, NAT 350s, 28k/60s≈466/s, 127s SYN math, ndots:5, PQC X25519MLKEM768 + ClientHello ossification, 47-day cert schedule), 1 partial (sharpened), 0 hard deltas.
- Biggest baseline gap: HTTP/3 trajectory — Opus says "steadily upward"; reality as of mid-2026 is a plateau with Cloudflare's edge share peaking ~2023 and declining, driven by QUIC losing to TCP kernel offloads on fast clean links.
- Kept as likely-delta despite unprobed: conntrack ceiling, retry multiplication across layers, gRPC pinning/MAX_CONNECTION_AGE, rightmost-XFF rule, JVM DNS caching, and the signature→prior table as a lookup artifact.
