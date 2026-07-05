---
name: api-design
description: Load when designing or reviewing any API surface — REST/HTTP endpoints, RPC or GraphQL schemas, library/public function signatures, SDKs, webhooks, or when evolving an existing API (versioning, deprecation, backwards compatibility). Also load when choosing between REST, RPC, and GraphQL, or designing error payloads, pagination, or idempotency.
---

# API Design

This is a correction sheet, not a survey. Standard API-design consensus (cursor pagination, RFC 9457 errors, idempotency keys, Deprecation/Sunset headers, webhook signing, PUT-vs-PATCH, 429 + Retry-After) is compressed to bare checklists below — you already know the reasoning. The corrections section is where the reflex answer is wrong; weight it accordingly.

## Corrections — where the reflex answer is wrong

1. **`required→optional` is breaking too, not just `optional→required`.** The reflex is "loosening a requirement is safe." It fails twice:
   - *Response side (the crisp case):* clients coded against "this field is always present" crash on absence. Removing `required` from a response property is flagged as breaking by compat linters (`oasdiff`) for exactly this reason.
   - *Request side:* integrations use your validation as their backstop. Requests that used to fail loudly now silently write partial data.
   Treat requiredness changes in **either direction** as major-version material. Start fields optional-with-documented-default; enforce later only at a major.

2. **Tightening validation doesn't just 400 new requests — it bricks existing records.** The commonly stated cost is "inputs we used to accept now get rejected." The worse, usually-missed shape: data stored under the old rules becomes permanently un-updatable — a client fetches a record with a legacy invalid email, edits one *unrelated* field, and the save fails, forever. Correction: grandfather stored data, or validate only the fields being changed — never re-validate the whole record on every write. (Same trap when a "bugfix" tightens a regex or shortens a max length.)

3. **"Is it a resource?" arguments answer themselves: it isn't.** If the team is debating whether `reserveInventory` is a resource, stop contorting — use RPC semantics. The custom-method form inside REST (`POST /things/{id}:archive`) is a legitimate escape hatch; the fake noun (`POST /thing-archival-requests`) is not clearer, it's just a worse RPC.

4. **A timeout does not mean the operation failed — it means the client doesn't know.** Design every non-idempotent mutation assuming the client will retry after a timeout; that's why idempotency keys are part of v1 design, not later hardening. Same logic makes silent buffering behind a synchronous name (`send()` that can drop data on process exit) a contract lie: name it `enqueue()`, ship `flush()`/`close()`.

5. **Default changes are breaking changes with extra stealth.** The schema diff is empty, so review passes — but flipping `page_size` 20→100 or `match=sensitive→insensitive` changes behavior for exactly the callers who didn't pass the parameter, which is most of them (that's why it was a default). Version or announce default flips like removals; a compat linter won't catch them.

## Compressed consensus (checklists — completeness is the point)

**Breaking regardless of label:** remove/rename any field, endpoint, or enum value · change type/format/units · tighten validation · requiredness change in either direction · default change · status/error-code change clients branch on · widen an enum you *return* (unless "handle unknown" + `UNKNOWN` sentinel shipped day one; proto: field 0 = `_UNSPECIFIED`, reserve deleted numbers) · reorder/reuse proto field numbers.

**Standard patterns (don't reinvent):**
- Pagination: opaque cursor of `(sort_key, id)`, absent token = end, max page size enforced. Offset = O(offset) + skips/dupes under writes.
- Unknown filter params → 400 with `code: UNKNOWN_PARAMETER`. Silently ignoring `?staus=active` returns *everything* and the client acts on it (mass-mail / data-exposure incident shape).
- Idempotency-Key: store `key → (request_hash, response)`, TTL ≥ 24h; same key + same body → replay stored response; same key + different body → 409/422, never re-execute.
- Errors: RFC 9457 shape; stable SCREAMING_SNAKE `code` (status alone can't split "bad email" from "quota exceeded"); `message` documented unstable; `request_id`; never leak stack/SQL/hostnames.
- Mutable resources: `ETag` + `If-Match`, or two read-modify-writers silently clobber.
- Rate limits: 429 + `Retry-After` + `RateLimit-*` headers, documented. Never shed load with 500 — clients retry 5xx and amplify the overload.
- Long ops (>~10s): `202` + operation resource, poll `GET /operations/{id}`; never hold a request past ~30s.
- Webhooks: HMAC signature, event `id` (dedup — duplicates are guaranteed), timestamp, "ordering not guaranteed" in docs, plus a polling `GET` to reconcile missed events. GraphQL: depth limits + query cost analysis in initial design, or you shipped a DoS endpoint.
- DELETE idempotent (second call → 404/204, never a special-case error); document soft- vs hard-delete visibility (does GET 404? can the ID be re-created?).
- By fiat: RFC 3339 UTC timestamps; money = integer minor units + currency code; IDs = prefixed opaque strings (`usr_9f3k2m`) documented opaque with max length.
- Always envelope: `{"users": [...]}` / response message in RPC — bare top-level arrays leave nowhere for `next_page_token` and can't evolve.
- PUT = full replace; PATCH must define "clear this field" explicitly (merge-patch collapses null/absent — proto uses `update_mask`).
- Response fields: decide required-always / optional-with-meaning / don't-ship-yet. Nullable-everything is a permanent null-check tax on every consumer.

**Library vs network, one-line physics:** library breaks are loud and local (build time, consumer's schedule; semver; exceptions and Big-O are part of the signature; never expose dependency types — their major version becomes yours). Network breaks are silent and global (runtime, your pager; every version runs forever; deprecate via headers + per-consumer telemetry + brownouts; field presence — null vs absent — is contract; don't leak DB schema as response schema).

**Deprecation order:** schema mark → per-consumer call telemetry (you can't remove what you can't measure) → `Deprecation`/`Sunset` headers → dated announcement + direct contact with top consumers → brownouts for the stragglers → 410.

**Signatures:** invalid states unrepresentable beats documented rules — enum over magic strings, one mode field over two exclusive booleans, one list of pairs over two same-length lists, format-specific options living *on* the format object. Booleans past ~2 → keyword-only/enum/config object. Names must imply cost, side effects, blocking, and mutation; one dishonest name fails the review.

## Self-check before presenting an API design

- Write the client code for the top 3 tasks *before* freezing the schema. >2 calls or client-side joins for a common task = wrong resource boundaries. This catches more than any checklist.
- Diff compat mechanically — `buf breaking` (protobuf), `oasdiff` (OpenAPI) — never eyeball it. Then hand-check the two things linters miss: default flips and behavior changes under unchanged schema.
- Simulate the four failure paths: retry-after-timeout (duplicate created?), write-during-pagination (skip/dupe?), never-seen enum value or `code` (client crash?), concurrent read-modify-write (clobber?).
- "What breaks when I remove this?" per field/method — if you can't defend the future removal cost, cut it now. Adding later costs one release; removing costs a deprecation cycle plus broken users.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 questions covering 14 claims: 12 baseline (cut/compressed), 1 partial (sharpened), 1 delta (expanded).
- Biggest gap: Opus grades `required→optional` as safe ("old clients still send it") — the both-directions-breaking rule is this skill's highest-value line.
- Partial: Opus knows tightening validation is breaking but misses the bricked-existing-records shape and the grandfather/validate-only-changed-fields correction.
- Everything else (pagination, idempotency, RFC 9457, deprecation incl. brownouts/410, webhooks, rate-limit shedding, bare arrays, PUT/PATCH/update_mask, lib-vs-network, signature pitfalls, client-code-first) Opus produced cold at or above the skill's specificity — retained only as stripped checklist lines.
