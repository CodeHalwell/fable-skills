---
name: api-design
description: Load when designing or reviewing any API surface — REST/HTTP endpoints, RPC or GraphQL schemas, library/public function signatures, SDKs, webhooks, or when evolving an existing API (versioning, deprecation, backwards compatibility). Also load when choosing between REST, RPC, and GraphQL, or designing error payloads, pagination, or idempotency.
---

# API Design

## Core mental model

- **An API is a promise you can't take back.** Every observable behavior — not just the documented one — will be depended on (Hyrum's Law). Design as if removal is impossible, because in practice it is. Budget design effort per expected consumer-year, not per implementation hour.
- **The best API is the one that can't be misused.** Prefer making invalid states unrepresentable over documenting valid usage:
  - A parameter that must be one of three strings should be an enum type.
  - Two booleans that can't both be true should be one mode field.
  - A function that must be called after `init()` should not exist — fold init into it, or have `init()` return the handle that exposes the method.
  - A pair of arguments that must be the same length should be one list of pairs.
- **Minimal surface area wins by default.** Every public name is a liability with compounding interest. Ship the 3 methods people need, not the 12 they might want. Adding later costs one release; removing costs a deprecation cycle plus broken users. When in doubt, leave it out.
- **Naming is the contract.** `get_user()` must not do network retries with 30s timeouts; `delete()` must not soft-delete unless the name says so; a function called `parse` must not also validate business rules or write to disk. If you can't name it honestly, the design is wrong, not the name.
- **Design for the reader of the call site, not the implementer.** Judge every signature by what calling code looks like in review with no docs open. `retry(op, RetryPolicy(max_attempts=3, backoff=Exponential(base_ms=100)))` reads; `retry(op, 3, 100, 2.0, True)` doesn't.
- **Consistency beats local optimality.** A slightly worse pattern used everywhere is a better API than the perfect pattern used once — consumers amortize learning across the whole surface. Match the platform's conventions (HTTP semantics, the language's stdlib idioms) before inventing.

## Decision frameworks

### REST vs RPC vs GraphQL
| Situation | Choose | Because |
|---|---|---|
| Public API, resource-shaped domain (things with IDs, CRUD-ish lifecycle) | REST | Uniform interface means clients guess correctly; HTTP caching, status codes, and tooling come free |
| Internal service-to-service, action-shaped domain (`reserveInventory`, `recomputeScore`) | RPC (gRPC/Connect) | Don't contort verbs into fake resources; typed stubs, streaming, and codegen beat hand-rolled REST clients |
| Many heterogeneous frontends with divergent data needs over a shared graph | GraphQL | Solves N over/under-fetch problems once; but you inherit N+1 resolvers, query cost limiting, and cache complexity — don't pick it for one frontend |
| Long-running operations (>~10s) | Any + operation resource | Return `202 Accepted` + operation ID immediately; client polls `GET /operations/{id}`. Never hold a request open past ~30s |
| Server→client notification | Webhooks + polling fallback | Always pair them: webhooks get dropped, endpoints go down; consumers need a `GET` to reconcile missed events |
| Bulk operations | Explicit batch endpoint with per-item results | Looping unary calls hits rate limits and N×RTT; batch responses must report per-item success/failure, not all-or-nothing |

Rule of thumb: if you're arguing about whether something "is a resource," it isn't — use RPC semantics. Custom-method style within REST (`POST /things/{id}:archive`) is a legitimate escape hatch; a fake resource (`POST /thing-archival-requests`) is not clearer.

### Evolution rules (network APIs)
- **Additive-only, forever.** Safe changes:
  - New optional request field (with a default that preserves old behavior).
  - New response field (if clients were told from day one to ignore unknown fields).
  - New endpoint/method; new enum value in requests you *accept*.
- **Breaking changes** (regardless of how they're labeled):
  - Removing or renaming any field, endpoint, or enum value.
  - Changing a field's type, format, or units; changing default behavior.
  - Tightening validation; making an optional field required.
  - Changing error codes or status codes clients branch on.
  - Reordering/renumbering protobuf fields; reusing a deleted field number.
- **Enum widening is a one-way trap.** Adding a value to an enum you *return* breaks every client with exhaustive matching. Either document "unknown values must be handled" from v1 day one and ship an `UNKNOWN` sentinel, or never widen returned enums. In protobuf, always reserve field 0 as `_UNSPECIFIED`.
- **optional→required is always breaking. required→optional is breaking too** — clients depend on the server rejecting bad input, and on the field always being present in responses. Start optional-with-default; enforce later only at a major version.
- **Deprecation mechanics, in order:**
  1. Mark in schema (`deprecated: true` in OpenAPI, `[deprecated = true]` in proto, `@deprecated` in GraphQL).
  2. Add telemetry counting calls per consumer — you cannot remove what you cannot measure.
  3. Signal in responses: `Deprecation` and `Sunset` headers (REST), warnings in payload metadata.
  4. Announce a removal date at least one client release cycle out; contact the top consumers directly.
  5. Brownout before removal: deliberate short failure windows surface the stragglers that ignored every email.
- **Version placement:** URL path for REST (`/v2/`), package for protobuf (`myapi.v2`). Bump major only for actual breaks. A v2 is a migration project for every client — batch years of breaks into one, or better, design so you never need it.

### Library API vs network API — different physics
| Concern | Library | Network |
|---|---|---|
| Compat unit | Compile/link: signatures, types, exceptions, *and* behavior | Wire: field names, types, status codes |
| Errors | Typed exceptions / result types; caller catches specific types | Error payload schema; caller branches on machine-readable `code` string |
| Versioning | Semver; breaking = major; users pin and upgrade deliberately | You run every version simultaneously; old clients never upgrade |
| Deprecation lever | Compiler warnings at build time | Headers + telemetry + brownouts at run time |
| Killer mistake | Exposing internal types (accepting a `requests.Session`, returning an ORM model) — now their API is your API | Leaking DB schema as response schema — now you can't refactor storage |
| Performance contract | Big-O and blocking behavior are part of the API (a `get()` that lazily makes a network call violates its name) | Latency/timeout expectations; documented idempotency so clients can retry |
| Extra rule | Exceptions thrown are part of the signature — swapping `ValueError` for a custom error is breaking | Field *presence* is part of the contract — `null` vs absent must be defined |

### Standard REST patterns (don't reinvent)
- **Pagination:** cursor-based. `?page_token=...&page_size=50` → `{"items": [...], "next_page_token": "..."}`.
  - Offset pagination breaks under concurrent writes (rows shift → items skipped or duplicated) and is O(offset) in most databases.
  - Make cursors opaque (base64 of `(sort_key, id)`), never raw offsets — clients will forge them if they can read them.
  - Absent/empty `next_page_token` = last page. Enforce a max `page_size` server-side.
- **Filtering/sorting:** explicit whitelisted params (`?status=active&order_by=created_at desc`). Reject unknown filter params with 400 — silently ignoring a typo'd filter (`?staus=active`) returns *everything*, and the client acts on it (the classic mass-mailing incident shape).
- **Idempotency keys:** any non-idempotent mutation (`POST /payments`) accepts an `Idempotency-Key` header.
  - Server stores `key → (request_hash, response)` with TTL ≥ 24h.
  - Replay with same key + same body: return the stored response, don't re-execute.
  - Same key + different body: return 409/422 — this is a client bug, never a re-execute.
  - Without this, clients that retry on timeout double-charge. A timeout does *not* mean the operation failed; it means the client doesn't know.
- **Error payload:** RFC 9457 (`application/problem+json`) shape or equivalent:
  - One machine-readable, stable `code` per failure kind (SCREAMING_SNAKE string). HTTP status alone can't distinguish "bad email" from "quota exceeded" — both are 4xx.
  - Human `message` explicitly documented as unstable (clients that parse it break on wording changes).
  - Optional `details` array for field-level validation errors; a correlation `request_id` for support.
  - Never leak stack traces, SQL, or internal hostnames.
- **Concurrency control:** for mutable resources, return `ETag` and honor `If-Match` on writes — otherwise two clients doing read-modify-write silently clobber each other and you'll retrofit it after a data-loss ticket.
- **Rate limiting as contract, not afterthought:** document the limits, return 429 with `Retry-After`, and expose remaining-quota headers (`X-RateLimit-Remaining` or the standard `RateLimit-*` set). An undocumented limit is a mystery outage from the client's perspective; a documented one is a design parameter they build around. Never rate-limit by returning 500 — clients retry 5xx, amplifying the overload you're shedding.
- **Deletion semantics:** `DELETE` must be idempotent — a second DELETE of the same resource returns 404 or 204, never an error the client must special-case. Decide and document soft- vs hard-delete: if soft, does the ID still 404 on GET? Can it be re-created? Ambiguity here surfaces as client-side data-model corruption.

## Failure modes & pitfalls

- **Boolean parameters metastasize.** `create_user(name, True, False, True)` — nobody can read the call site, and each new flag doubles the config space. Correction: keyword-only arguments in Python (`def create_user(name, *, verified=False)`), enums for modes, a config object past ~3 options.
- **Returning naked collections.** `GET /users → [...]` leaves nowhere to put `next_page_token` or `total_count` later — adding an envelope then is a breaking change. Always return an object: `{"users": [...]}`. Same in RPC: never return a bare list or scalar; wrap in a response message.
- **`PUT` with partial semantics.** Implementing PUT as merge means a client sending the full object can't clear a field. PUT = full replace; partial update = PATCH. For PATCH, decide explicitly how "clear this field" is expressed — JSON merge-patch can't distinguish "absent" from "set to null" in every language's deserializer, which is why protobuf APIs use an explicit `update_mask`.
- **200 with an error in the body.** Breaks every retry policy, monitor, and cache between you and the client. Status codes are contract: 4xx = caller's fault, don't retry unchanged; 5xx = yours, retry with backoff; 429 = include `Retry-After`. Corollary: don't return 500 for validation failures — clients will retry a request that can never succeed.
- **Tightening validation as a "bugfix."** You start rejecting emails without TLDs; clients who stored such data can now never update those records (fetch → modify one unrelated field → save fails on the old email). Validation tightening is breaking. Grandfather existing data, or validate only fields being changed.
- **Timestamps and money as local conventions.** Epoch seconds vs millis confusion, naive datetimes, floats for currency. Fix by fiat: RFC 3339 UTC strings (`2026-07-04T12:00:00Z`) for timestamps; integer minor units plus currency code (`{"amount": 1999, "currency": "USD"}`) for money. Floats for money is an instant credibility fail.
- **IDs that leak and get parsed.** Sequential integers leak business volume and invite enumeration attacks; clients *will* parse structured IDs (`user_12` → split on `_`) and break when the format changes. Use prefixed opaque strings (`usr_9f3k2m`), document them as opaque with a max length, never reuse them.
- **The god endpoint / god function.** `?include=orders,orders.items,profile` or `process(data, mode, options)` — one name whose behavior forks internally on parameters. Each mode has different performance, permissions, and failure semantics; you can never change one without auditing all. Split by use case.
- **Async fire-and-forget behind a synchronous name.** A client library whose `send()` buffers internally and can drop data on process exit must make that visible: name it `enqueue()`, provide `flush()`/`close()`, document delivery semantics. Silent at-most-once behind a name like `send` misleads everyone downstream.
- **Exposing your dependency's types in your signatures.** Accepting/returning `pandas.DataFrame`, an ORM model, or a `boto3` client in a public library API welds your major version to theirs and blocks callers who don't use that dependency. Accept protocols/plain data at the boundary; convert internally.
- **Designing v1 without writing a v1 client.** Write the client code for your top 3 use cases *before* freezing the schema. If a common task takes 3 calls plus client-side joins, the resource boundaries are wrong. This one exercise catches more design flaws than any review checklist.
- **GraphQL without cost control.** Shipping a public GraphQL endpoint without depth limits, query cost analysis, and (for known clients) persisted queries hands out a DoS endpoint: `{ users { friends { friends { friends {...}}}}}`. This is part of initial design, not later hardening.
- **Webhooks without signing, ordering, and replay rules.** Consumers need: an HMAC signature header to verify sender, an event `id` for dedup (you *will* deliver duplicates), a timestamp, and a documented statement that ordering is not guaranteed. Omit any of these and every consumer builds a different wrong workaround.
- **Treating defaults as free to change.** Flipping a default (`page_size` 20→100, timeout 30s→10s, `match=sensitive→insensitive`) changes behavior for every caller who didn't pass the parameter — which is most of them, which is why it was a default. Default changes are breaking changes with extra stealth; version them or announce them like removals.
- **Nullable everything.** Marking every response field nullable "to be safe" pushes a null-check tax into every consumer forever and hides real optionality signals. Decide per field: required-always (document it, never null), optional-with-meaning (document what absence means), or don't ship the field yet.

## Worked micro-examples

### 1. Evolving a search endpoint without breaking anyone
v1 ships: `GET /v1/products?q=term` → `{"products": [...], "next_page_token": "..."}`.
Requirement: add category filtering and relevance scores; also `q` matching was accidentally case-sensitive and users want it case-insensitive.

1. **Add `?category=` as optional.** Absent = old behavior. Reject `?catagory=` (unknown param) with 400 + `code: "UNKNOWN_PARAMETER"` — you'll be grateful the first time someone typos a filter that would otherwise silently match everything.
2. **Add `relevance_score` to each product.** New response fields are safe *if* v1 docs said "ignore unknown fields." If they didn't, canary the rollout and watch client error telemetry — some strict deserializer (a client with `DisallowUnknownFields` on) will break, and that's your problem now regardless of fault.
3. **The case-sensitivity fix is behavior-breaking even though it's "a bug."** Someone depends on the current behavior (Hyrum). Loosening match rules changes result sets under existing queries. Ship behind `?match=insensitive`, announce the default flip with a `Deprecation` header and dated changelog, flip after the window, keep `?match=sensitive` as an escape hatch. Cost: one parameter. Cost of "just fix it": unexplained result-set changes in every dependent system, discovered in production.
4. **What you don't do:** bump to `/v2` (this is all additive); overload `q` with a mini-language (`q=category:tools term` — an unversionable, unparseable contract); return scores only when a flag is set (forked response shapes double every client's test matrix).

### 2. Library signature review, before/after
```python
# Before: misusable
def export(data, path, fmt="csv", compress=False, overwrite=False,
           header=True, sep=","):  # sep meaningless unless fmt="csv"
    ...
# After: invalid states unrepresentable, call sites readable
@dataclass(frozen=True)
class Csv:  sep: str = ","; header: bool = True
@dataclass(frozen=True)
class Parquet: pass

def export(data, path: Path, format: Csv | Parquet = Csv(), *,
           compress: bool = False,
           if_exists: Literal["error", "overwrite"] = "error") -> ExportReport:
    ...
```
The moves: format-specific options live *on* the format (can't pass `sep` with Parquet); booleans become keyword-only; `overwrite=False` becomes a named policy that can grow (`"append"`) without a new flag; a structured return replaces `None` so success is inspectable. Each move removes a documented rule by making it a type rule.

## Self-check before presenting an API design

- Write the client code for the 3 most common tasks. Any task needing >2 calls, client-side joins, or a comment to explain a parameter → redesign that part.
- For each field/param/method, ask "what breaks when I remove this?" — if you can't defend keeping it against that future cost, cut it now.
- Diff against the previous version mechanically: any removed/renamed/retyped field, tightened validation, changed default, changed status code, or widened *response* enum is a breaking change no matter how it's labeled. Use a schema-compat linter (`buf breaking` for protobuf, `oasdiff` for OpenAPI) rather than eyeballing.
- Simulate the failure paths end to end:
  - Client times out mid-POST and retries → duplicate created? Needs idempotency key.
  - Data mutates mid-pagination → items skipped/duplicated? Needs cursors.
  - Server returns a `code` or enum value the client has never seen → client crash? Needs documented unknown-handling.
  - Two clients read-modify-write the same resource → silent clobber? Needs ETag/If-Match.
- Check every name against its behavior: side effects, blocking, mutation, and cost must all be implied by the name or signature. One dishonest name fails the review.
- Confirm error responses carry a stable machine-readable `code`, and that nothing in any payload leaks internals (stack traces, SQL, hostnames, sequential IDs).
