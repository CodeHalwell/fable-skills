---
name: api-design
description: Load when designing or reviewing any API surface — REST/HTTP endpoints, RPC or GraphQL schemas, library/public function signatures, SDKs, webhooks, or when evolving an existing API (versioning, deprecation, backwards compatibility). Also load when choosing between REST, RPC, and GraphQL, or designing error payloads, pagination, or idempotency.
---

# API Design

## Core mental model

- **An API is a promise you can't take back.** Every observable behavior — not just the documented one — will be depended on (Hyrum's Law). Design as if removal is impossible, because in practice it is: budget one design hour per expected consumer-year, not per implementation hour.
- **The best API is the one that can't be misused.** Prefer making invalid states unrepresentable over documenting valid usage. A parameter that must be one of three strings should be an enum; two booleans that can't both be true should be one mode field; a function that must be called after `init()` should not exist — fold init into it or return a handle from init.
- **Minimal surface area wins by default.** Every public name is a liability with compounding interest. Ship the 3 methods people need, not the 12 they might want. You can always add; you can never remove. When in doubt, leave it out — the cost of adding later is one release; the cost of removing is a deprecation cycle plus broken users.
- **Naming is the contract.** `get_user()` must not do network retries with 30s timeouts; `delete()` must not soft-delete unless the name says so; a function called `parse` must not also validate business rules. If you can't name it honestly, the design is wrong, not the name.
- **Design for the reader of the call site, not the implementer.** Judge every signature by what the calling code looks like in a code review with no docs open. `retry(op, RetryPolicy(max_attempts=3, backoff=Exponential(base_ms=100)))` reads; `retry(op, 3, 100, 2.0, True)` doesn't.

## Decision frameworks

### REST vs RPC vs GraphQL
| Situation | Choose | Because |
|---|---|---|
| Public API, resource-shaped domain (things with IDs, CRUD-ish lifecycle) | REST | Uniform interface = clients guess correctly; HTTP caching, status codes, and tooling come free |
| Internal service-to-service, action-shaped domain (`reserveInventory`, `recomputeScore`) | RPC (gRPC/Connect) | Don't contort verbs into fake resources (`POST /inventory-reservations` for a transient action); typed stubs and streaming beat hand-rolled REST clients |
| Many heterogeneous frontends with divergent data needs over a shared graph | GraphQL | Solves N over/under-fetch problems once; but you inherit N+1 resolvers, query cost limiting, and cache complexity — don't pick it for one frontend |
| Long-running operations | Any + operation resource | Return `202` + operation ID immediately; never hold a request open past ~30s |
| Server→client notification | Webhooks + polling fallback | Always pair: webhooks get dropped; consumers need a `GET` to reconcile |

Rule of thumb: if you're arguing about whether something "is a resource," it isn't — use RPC semantics (`POST /things/{id}:archive` custom-method style is fine within REST).

### Evolution rules (network APIs)
- **Additive-only, forever.** Safe: new optional request field, new response field, new endpoint, new enum value *in requests you accept*. Breaking: removing/renaming anything, changing a type, tightening validation, making optional required, changing default behavior, changing error codes clients branch on, reordering/renumbering protobuf fields.
- **Enum widening is a one-way trap.** Adding a value to an enum you *return* breaks every client with exhaustive matching. Either document "unknown values must be handled" from v1 day one and ship an `UNKNOWN` sentinel, or never widen returned enums. In protobuf, always keep field 0 as `_UNSPECIFIED`.
- **optional→required is always breaking; required→optional is breaking too** (clients depending on the server rejecting bad input, and on the field being present in responses). Start optional-with-default; you can enforce later only at a major version.
- **Deprecation mechanics:** mark in schema (`deprecated: true`, `@deprecated`, `[[deprecated]]`), emit telemetry counting callers per consumer, warn in responses (`Deprecation` + `Sunset` headers), set a date ≥ one client release cycle out, then *brownout* (deliberate temporary failures) before removal — silent removal after a doc note strands the long tail every time.
- **Version in the URL or media type for REST (`/v2/`), package name for protobuf (`myapi.v2`).** Only bump major for actual breaks. A v2 is a migration project for every client — batch years of breaks into one, or better, never need it.

### Library API vs network API — different physics
| Concern | Library | Network |
|---|---|---|
| Compat unit | Compile/link: signatures, types, exceptions, *and* behavior | Wire: field names, types, status codes |
| Errors | Typed exceptions / result types; caller catches specific types | Error payload schema; caller branches on machine-readable `code` string |
| Versioning | Semver; breaking = major, users pin | You run every version simultaneously; old clients never upgrade |
| Killer mistake | Exposing internal types (accepting a `requests.Session`, returning an ORM model) — now their API is your API | Leaking DB schema as response schema — now you can't refactor storage |
| Performance contract | Big-O and blocking behavior are part of the API (a `get()` that lazily makes a network call violates the name) | Latency/timeout expectations; document idempotency so clients can retry |

### Standard REST patterns (don't reinvent)
- **Pagination:** cursor-based (`?page_token=...&page_size=50` → `{items, next_page_token}`). Offset pagination breaks under concurrent writes (rows shift → skipped/duplicated items) and is O(offset) in most DBs. Make cursors opaque (base64 of `(sort_key, id)`), never raw offsets — clients will forge them. Absent `next_page_token` = last page.
- **Filtering/sorting:** explicit whitelisted params (`?status=active&order_by=created_at desc`). Reject unknown filter params with 400 — silently ignoring a typo'd filter (`?staus=active`) returns *everything* and the client acts on it (the classic mass-mailing incident shape).
- **Idempotency keys:** any non-idempotent mutation (`POST /payments`) accepts `Idempotency-Key`. Server stores `key → (request_hash, response)` with TTL ≥ 24h; replay with same key+body returns the stored response; same key+different body returns `409`/`422`. Without this, clients that retry on timeout double-charge — a timeout does *not* mean the operation failed.
- **Error payload:** one machine-readable stable `code` (SCREAMING_SNAKE string, not just HTTP status — 400 alone can't distinguish "bad email" from "quota exceeded"), human `message` explicitly marked unstable, optional `details` array for field-level errors, and a correlation `request_id`. RFC 9457 (`application/problem+json`) is the standard shape. Never leak stack traces, SQL, or internal hostnames.

## Failure modes & pitfalls

- **Boolean parameters metastasize.** `create_user(name, True, False, True)` — nobody can read the call site, and the next flag doubles the config space. Correction: keyword-only args in Python (`def create_user(name, *, verified=False)`), enums for modes, or a config object past ~3 options.
- **Returning naked collections.** `GET /users → [ ... ]` leaves nowhere to put `next_page_token` or `total_count` later — adding an envelope is a breaking change. Always return an object: `{"users": [...]}`. Same for RPC responses: never return a bare list or scalar.
- **`PUT` with partial semantics.** Implementing PUT as merge-patch means a client sending the full object can't clear a field. PUT = full replace; partial update = `PATCH`. For PATCH, decide explicitly how "clear this field" is expressed (JSON `null` vs field mask) — JSON merge-patch can't distinguish "absent" from "set to null" in every language's deserializer, which is why protobuf APIs use explicit `update_mask`.
- **200 with an error in the body.** Breaks every retry policy, monitor, and cache between you and the client. Status codes are part of the contract: 4xx = caller's fault, don't retry unchanged; 5xx = yours, retry with backoff; 429 = include `Retry-After`. Corollary: don't return 500 for validation failures — clients will retry a request that can never succeed.
- **Tightening validation as a "bugfix."** You start rejecting emails without TLDs; clients who stored such data can now never update those records (fetch → modify one field → save fails on the old email). Validation tightening is breaking. Grandfather existing data or validate only changed fields.
- **Timestamps and money as local conventions.** Epoch seconds vs millis, naive datetimes, floats for currency. Fix by fiat: RFC 3339 UTC strings for timestamps (`2026-07-04T12:00:00Z`), integer minor units + currency code for money (`{"amount": 1999, "currency": "USD"}`). Floats for money is an instant expert-credibility fail.
- **IDs that leak and get parsed.** Sequential integer IDs leak volume and invite enumeration; clients *will* parse structured IDs (`user_12` → split on `_`) and break when the format changes. Use prefixed opaque strings (`usr_9f3k2m`), document "opaque, ≤ N chars," and never reuse.
- **The god endpoint / god function.** `?include=orders,orders.items,profile` or `process(data, mode, options)` — one name whose behavior forks internally on parameters. Each mode has different perf, permissions, and failure modes; you can never change one without auditing all. Split by use case.
- **Async fire-and-forget defaults in libraries.** A client library whose `send()` buffers internally and can drop data on process exit must make that visible: name it `enqueue()`, provide `flush()`/`close()`, and document delivery semantics. Silent at-most-once behind a name like `send` misleads everyone.
- **Designing v1 without a v1 client.** Write the client code for your top 3 use cases *before* freezing the schema. If a common task takes 3 calls plus client-side joins, the resource boundaries are wrong. This one exercise catches more design flaws than any review.
- **GraphQL without cost control.** Shipping a public GraphQL API without depth limits, query cost analysis, and persisted queries hands out a DoS endpoint (`{ users { friends { friends { friends ... }}}}`). Not optional hardening — part of the initial design.

## Worked micro-example: evolving a search endpoint without breaking anyone

v1 ships: `GET /v1/products?q=term` → `{"products": [...], "next_page_token": "..."}`.

Requirement: add category filtering and relevance scores; also, `q` matching was accidentally case-sensitive and users want it case-insensitive.

Expert moves:
1. **Add `?category=` as optional.** Absent = old behavior. Reject `?catagory=` (unknown param) with 400 + `code: "UNKNOWN_PARAMETER"` — you're grateful for this the first time someone typos a filter that would otherwise silently match everything.
2. **Add `relevance_score` to each product object.** New response fields are safe *if* v1 docs said "clients must ignore unknown fields." If v1 never said that, canary the change and watch client error telemetry before full rollout — some strict deserializer (e.g., a client using `DisallowUnknownFields`) will break, and that's now your problem regardless of fault.
3. **Case-sensitivity fix is behavior-breaking even though it's "a bug."** Someone depends on it (Hyrum). Loosening match rules changes result sets under existing queries. Ship behind `?match=insensitive`, announce default flip with a `Deprecation` header and a dated changelog entry, flip after a deprecation window, keep `?match=sensitive` as escape hatch. Cost: one extra param. Cost of the "just fix it" route: unexplained result-set changes in every dependent system, discovered in production.
4. **What you don't do:** bump to `/v2` (this is all additive), overload `q` with a mini-language (`q=category:tools term` — unversionable, unparseable contract), or return scores only when a flag is set (forked response shapes double client test matrix).

## Self-check before presenting an API design

- Write the client code for the 3 most common tasks. Any task needing >2 calls, client-side joins, or a comment to explain a parameter → redesign that part.
- For each field/param/method, ask "what breaks when I remove this?" — if you can't defend keeping it against that future cost, cut it now.
- Diff against previous version mechanically: any removed/renamed/retyped field, tightened validation, changed default, changed status code, or widened *response* enum → it's a breaking change no matter how it's labeled. Run a schema-compat linter (e.g., `buf breaking` for protobuf, `oasdiff` for OpenAPI) rather than eyeballing.
- Simulate the failure paths: client times out mid-`POST` and retries (duplicate created? → need idempotency key), page of results mutates mid-pagination (items skipped? → need cursor), server returns a `code` the client has never seen (crash? → need documented unknown-handling).
- Check every name against its behavior: side effects, blocking, mutation, and cost must all be implied by the name or signature. One dishonest name fails the review.
