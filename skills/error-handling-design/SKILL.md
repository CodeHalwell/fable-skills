---
name: error-handling-design
description: Load when designing how code reports, propagates, or recovers from failures — choosing exceptions vs result types vs error codes, designing retry/idempotency semantics, structuring error chains and messages, deciding what to catch where, or reviewing error-handling code.
---

# Error Handling Design

## Core mental model

- **Errors are part of your API, designed with the same care as the happy path.** Every public function's contract includes: which failures it can report, how (type/value/exception), what state the world is in afterward, and whether retrying is safe. If you can't answer those four questions for a function you wrote, its error handling is undesigned, not "simple."
- **Classify every error into one of three kinds; each demands a different response:**
  - **Programmer error** (bug): violated precondition, impossible state, null where non-null was guaranteed. Response: fail fast and loudly — assert/crash/500 + page. *Never* catch-and-continue: the process state is now unproven, and handling bugs at runtime just moves the crash somewhere less diagnosable.
  - **Operational error** (environment): network down, disk full, timeout, dependency 503. Expected in a working system. Response: designed-for — retry, degrade, circuit-break, queue, surface. These are the errors your handling code exists for.
  - **User/domain error** (input or state rejection): validation failure, insufficient funds, not found, conflict. Not exceptional at all — a normal, frequent outcome. Response: report precisely to the caller (field-level detail, 4xx); never retry (same input → same result), never page anyone.
  - The commonest architectural error-handling mistake is one mechanism for all three: retrying validation errors, paging on user errors, "handling" bugs.
- **Handle errors at the layer that has the context to decide, propagate everywhere else.** Most code should not catch. The rare catch sites are: the top-level boundary (request handler, worker loop, main) which maps error kind → response and logs once; and specific mid-layers that can genuinely *recover* (retry with a fallback, substitute a default *that is correct*, not merely convenient).
- **Preserve the chain, log once.** An error's value is its diagnostic content: root cause, every context layer, stack. Each rethrow must wrap-with-context, not replace. Each error must be logged exactly once — at the layer that handles it. Log-and-rethrow at every layer produces five stack traces per incident and pages that no one can read.
- **Design for partial failure.** The interesting question is never "what if it fails?" but "what if it fails *after the first write and before the second*?" Every multi-step effect needs an answer: transaction, idempotency, compensation, or documented inconsistency.

## Choosing the reporting mechanism (per-ecosystem, not by taste)

Follow the ecosystem's native idiom — fighting it taxes every caller:

| Ecosystem | Idiom | Rules |
|---|---|---|
| Python | Exceptions | Raise specific subclasses of a per-library base (`class AppError(Exception)`); callers can `except AppError` without catching bugs like `TypeError`. Never raise bare `Exception`/`str`. Return-`None`-on-error is acceptable only for single obvious-absence lookups (`dict.get` style) — never for operations with multiple failure modes. |
| Go | `(T, error)` returns | Wrap with `%w` (`fmt.Errorf("loading user %d: %w", id, err)`); test with `errors.Is`/`errors.As`, never string matching. Sentinel errors (`ErrNotFound`) for conditions callers branch on; typed errors for ones carrying data. Panics only for programmer error. |
| Rust | `Result<T, E>` + `?` | `thiserror` for library error enums (callers match), `anyhow` for application glue (callers don't). `panic!`/`unwrap` = programmer error only; `.unwrap()` on external input is a bug. |
| Java/Kotlin | Unchecked exceptions dominate | Checked exceptions in modern practice: use sparingly or not at all (they leak through signatures and get wrapped-blindly). Kotlin: `Result`/sealed classes for domain outcomes, exceptions for operational. |
| JS/TS | Exceptions + rejected promises | The killer bugs: un-awaited promise → unhandled rejection escaping the `try` around it; `throw` of non-`Error` values (lose stack). Always `throw new Error`/subclass; lint `no-floating-promises`. |
| C | Error codes | Every call site checked (lint/warn-unused-result); a single errno-style channel plus out-params; document ownership on failure (who frees what when the call fails — half-constructed object leaks live here). |

Cross-cutting choice: use **result-style types for domain outcomes** even in exception languages when failure is frequent and expected — a parser returning `ParseResult` with error positions beats throwing on the first of 500 bad rows. Use exceptions for the operational layer beneath. The distinction: is failure part of the *answer* (result type) or a failure to *produce* an answer (exception)?

## Retry semantics and idempotency

- **Retry only operational errors, and only ones marked retryable.** Timeouts, 429, 502/503, connection reset: yes. 400/401/403/404, validation, business rejection: never — retrying deterministic failures adds load and delay for nothing. Classify at the source: your error types should carry `retryable` explicitly rather than every caller re-deriving it from status codes.
- **A timeout is not a failure — it's an unknown outcome.** The request may have succeeded after you stopped waiting. Therefore *retrying anything non-idempotent on timeout is a double-execution bug* (double charge, duplicate email). Before adding any retry, prove idempotency or add it.
- **Making operations idempotent:** natural idempotency (SET x=5, PUT full-resource) when you can; otherwise **idempotency keys** — client generates a UUID per logical operation, server stores `(key → result)` with a TTL comfortably exceeding the retry horizon and returns the stored result on replay. The key must be *stored atomically with the effect* (same DB transaction / unique constraint on the key column) — checking-then-writing the key separately just narrows the duplicate window, doesn't close it.
- **Retry mechanics:** exponential backoff + **full jitter** (`sleep(random.uniform(0, base * 2**attempt))`) — synchronized backoff without jitter produces coordinated retry storms that re-kill a recovering dependency. Cap attempts (2–3 is usually right) and cap total elapsed time below the caller's own timeout, or your retries outlive the request they serve. **Never stack retries across layers** (client retries × service retries × proxy retries = 3×3×3 = 27 requests from one user click): retry at one designated layer, pass through elsewhere.
- **Circuit breakers** where retries meet a hard-down dependency: after N consecutive failures, fail fast for a cooldown instead of adding timeout-latency to every request. Decide per call site what fail-fast returns: cached/stale data (fail open) or error (fail closed) — a security check fails closed; a recommendations widget fails open.

## Context preservation in error chains

- Wrap at each layer with what *that layer* knows: identifiers, operation, parameters. Python: `raise PaymentError(f"charging order {order_id}") from err` — the `from` keeps `__cause__` and both tracebacks. Losing the chain (`raise NewError(...)` with no `from`, or Go `fmt.Errorf` with `%v` instead of `%w`) converts every incident into archaeology.
- **Message discipline:** each message states its own layer's contribution only — the chain assembles the story. Anti-pattern: every layer prefixing "Error:" and restating the child ("Error: failed to process: error processing failed: ..."). Include the *values* that discriminate ("user 4211", "after 3 attempts", "endpoint /v2/charge"), not just the operation name.
- **Never put secrets or PII in error messages** — errors flow to logs, monitoring SaaS, and sometimes API responses. Redact tokens/card numbers at construction time, not at log time.
- Boundary translation: at API edges, map internal chains to a stable external shape — machine-readable `code` (string enum callers can switch on — never make clients parse prose), human `message`, optional `details`, and a `correlation_id` that links to the full internal chain in your logs. Internal stack traces in HTTP responses are an information leak *and* a compatibility trap (clients start matching on them — Hyrum's law applies to error strings).

## Crash-only thinking

- If your recovery path for corrupted in-process state is anything other than "exit and restart clean," you now have *two* code paths to keep correct, and the recovery path is the untested one. Prefer: supervisor restarts (systemd, k8s, Erlang-style), idempotent startup, durable state outside the process, and crash on programmer error. A process that can be safely killed at any instant is also a process that deploys, scales, and fails over safely — these are the same property.
- Consequences to actually implement: startup must handle the half-finished work of a predecessor (journals/WAL, at-least-once queues + idempotent consumers); "graceful shutdown" is an optimization, not a correctness requirement; any state that must survive crash goes to durable storage *at the moment it matters*, not at shutdown.
- Corollary for handlers: on detecting impossible state (checksum mismatch, invariant violation), the safe move is crash-and-restart, not best-effort repair — repair code running on state you don't understand makes corruption worse and unreproducible.

## When swallowing an error is correct

Swallowing (catch, don't propagate, continue) is right only when ALL hold: (1) the operation is genuinely optional to the caller's goal, (2) you substitute a *correct* fallback value or no-op, (3) you still record it (log/metric — at debug level if truly routine), (4) the catch is narrow (specific exception type, tight scope). Legitimate cases:
- Best-effort telemetry/cache-write/prefetch: failure to record a metric must never fail the request. `except MetricsError: metrics_dropped.inc()`.
- Cleanup during teardown: closing a connection that's already broken while handling the original error — suppress the secondary, preserve the primary (Python `contextlib.suppress(OSError)` around `conn.close()` in `finally`).
- Races that resolve themselves: `mkdir` → `FileExistsError` when the directory existing is the goal; deleting an already-deleted row.
- Last-resort isolation loops: a worker processing independent items catches *per item*, records, and continues — one poison message must not stop the queue (but must go to a dead-letter, not be dropped).

Illegitimate: `except Exception: pass` (catches bugs — the process continues with unproven state and the bug surfaces later, elsewhere); catching to avoid a red log line; swallowing in a library (the *application* decides policy, libraries propagate); returning a default that is merely type-correct (empty list from a failed fetch reads as "no results" — the user sees an empty inbox instead of an error, which is a lie).

## Failure modes and pitfalls

- **The catch-log-continue with wrong state:** `except Exception: log.error(...)` then falling through to code that assumes the try succeeded — variables unset or stale from the previous loop iteration. If you catch, the very next line must handle the *world where the try did not happen*.
- **Catching too broadly for one expected case:** wrapping 30 lines in `try/except ValueError` to guard one `int(s)` — a *different, buggy* ValueError 20 lines down is now silently classified as bad user input. Keep try bodies to the single statement that can legitimately fail.
- **Retry without rewind:** retrying a function whose first attempt already consumed the input (advanced a stream/iterator, popped from a queue, mutated the request body). Attempt 2 operates on different input. Retries must wrap a from-scratch closure over immutable inputs.
- **Error-path resource handling:** the error path is where leaks live — connection checked out, exception raised before release; response body never closed on non-200. Every acquire needs `with`/`defer`/`finally` that the *error* path also traverses; specifically check early-`return`s inside `try` before `finally`-less cleanup.
- **Exceptions across concurrency boundaries:** a task/future's exception is silently stored until awaited — fire-and-forget tasks vanish with their errors (Python: `asyncio.create_task` without holding a reference + done-callback; JS: floating promises). Every spawned task needs an owner that observes its result.
- **Exception in the error handler:** the logger/alerter itself throwing (serialization failure on the exotic object you attached) masks the original error. Boundary handlers wrap their own reporting in a last-ditch bare catch that writes something primitive (stderr) — the one place a broad catch is right.
- **errno/status shadowing in C-style code:** calling another function between failure and reading the error code; the second call overwrites it. Capture immediately.
- **Treating `finally`/`defer` as infallible:** cleanup that itself throws replaces the in-flight exception (Python pre-suppress; Go `defer f.Close()` on a *write* path ignores the flush error — data loss reported as success; check `Close()`'s error on writes).
- **Alert design errors:** paging on user errors (a bot probing 404s wakes a human) or *not* paging on error-rate change because "each error is handled." Handled ≠ healthy: alert on rates and budgets by error class, page only on operational/programmer classes.

## Worked micro-example: idempotent charge with correct retry

```python
class PaymentError(Exception): ...
class PaymentUnavailable(PaymentError): ...   # operational, retryable
class CardDeclined(PaymentError): ...          # domain, never retry

def charge(order_id: str, amount_cents: int, idem_key: str) -> ChargeResult:
    for attempt in range(3):
        try:
            resp = gateway.post("/charge",
                json={"amount": amount_cents, "idempotency_key": idem_key},
                timeout=(3.05, 10))                       # connect, read — never unbounded
        except (requests.ConnectionError, requests.Timeout) as err:
            if attempt == 2:
                raise PaymentUnavailable(f"charge {order_id}: gateway unreachable after 3 attempts") from err
            time.sleep(random.uniform(0, 0.4 * 2**attempt))   # full jitter
            continue
        if resp.status_code == 402:
            raise CardDeclined(f"charge {order_id}: {resp.json()['decline_code']}")  # no retry, no page
        if resp.status_code >= 500 or resp.status_code == 429:
            if attempt == 2:
                raise PaymentUnavailable(f"charge {order_id}: gateway {resp.status_code}") from None
            time.sleep(random.uniform(0, 0.4 * 2**attempt)); continue
        resp.raise_for_status()                     # 4xx here = OUR bug (bad request shape) → propagate as bug
        return ChargeResult.from_json(resp.json())
```
Load-bearing decisions: retrying timeout is safe *only because* `idem_key` (generated once by the caller per order, stored by the gateway) makes replay a read; 402 is a domain outcome typed so callers can branch and no one retries it; unexpected 4xx propagates as a programmer error rather than being retried or swallowed; the chain (`from err`) keeps the socket-level cause; total worst-case delay (~2.8s + timeouts) stays inside the caller's budget by construction.

## Verification / self-check

1. For each public function: name its failure modes, their kinds (bug/operational/domain), the reporting type, post-failure state, and retry-safety. Anything unnameable is undesigned.
2. Grep the diff for `except Exception`, `except:`, `catch (e) {}`, `_ = err`, `.unwrap()`, empty `catch` — each needs a justification meeting the four swallowing conditions.
3. Trace one error of each kind from raise site to human: is it logged exactly once, with chain intact, mapped to the right response (5xx+page / 4xx+field detail), with a correlation id?
4. For every retry: what proves the operation idempotent? What bounds attempts *and* total time? Is any other layer also retrying the same call?
5. Kill-test the partial-failure points: for each multi-step effect, simulate death between steps (or at least trace it on paper) and confirm restart converges — no stranded state, no double effect.
6. Confirm no error message contains secrets/PII and no external response contains internal stack frames.
