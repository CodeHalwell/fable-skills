---
name: error-handling-design
description: Load when designing how code reports, propagates, or recovers from failures — choosing exceptions vs result types vs error codes, designing retry/idempotency semantics, structuring error chains and messages, deciding what to catch where, or reviewing error-handling code.
---

# Error Handling Design

Your cold baseline in this domain is strong and should be trusted: the three error kinds (bug → crash loudly; operational → retry/degrade/circuit-break; user/domain → precise 4xx, never retry, never page), catch-at-the-decision-boundary and log exactly once, timeout = unknown outcome (never blind-retry non-idempotent ops), client-generated idempotency keys inserted atomically with the effect (unique constraint, same transaction), exponential backoff with full jitter (`sleep(uniform(0, base·2^attempt))`, ~3 attempts) at exactly one layer, circuit breakers with per-call-site fail-open/fail-closed, crash-only design (startup = recovery, kill-9-safe), RFC 9457 payloads with a stable machine-readable `code` + correlation id, and result-types-for-domain / exceptions-for-operational. Don't re-derive any of that — state the conclusion and move on. Below is only what the reflex answer gets wrong, plus the checklist for the gap between knowing these rules and applying them in written code.

## Corrections to reflex answers

- **Java: do not recommend checked exceptions "for recoverable conditions."** That's the JDK-1.x doctrine and it is still the reflex answer, but it's outdated. Modern Java practice is unchecked nearly everywhere: checked exceptions don't compose with lambdas/streams (`Stream.map` can't throw them), leak implementation detail through every intermediate signature, and empirically get wrapped blindly or swallowed. Default to specific `RuntimeException` subclasses of a per-module base; Kotlin dropped checked exceptions entirely for these reasons. If checked at all: only where the immediate caller must branch on the failure and there's a single call layer.
- **Error-swallowing policy belongs to the application, never a library.** The reflex catalogue of "legitimate swallowing" cases omits this: a library that catches-and-defaults (or catches-and-logs) has made a policy decision its callers cannot undo. Libraries propagate; only the application knows whether a dropped metric or stale cache is tolerable in its context.
- **Return-`None`-on-error is acceptable in Python only for a single obvious-absence lookup** (`dict.get` style). The moment an operation has two or more failure modes, `None` conflates them — raise typed exceptions or return a result object with the distinction intact.

## Applied-review checklist

These rules are all stated correctly when asked cold, then violated in written code — review against the list, don't trust that knowing implies doing.

1. Per public function: name its failure modes, their kinds, the reporting type, post-failure state, and retry-safety. Anything unnameable is undesigned, not "simple."
2. Grep the diff for `except Exception`, bare `except:`, `catch (e) {}`, `_ = err`, `.unwrap()`, empty catch — each must satisfy all four swallowing conditions: optional to the caller's goal, *domain-correct* fallback (an empty list from a failed fetch renders as "no results" — a lie), still recorded, narrowly scoped.
3. After every catch, the next line must handle the world where the try never ran — variables set inside the try, state stale from the previous loop iteration.
4. The *error* path must traverse every resource release: early `return` inside `try` without `finally`/`defer`/`with`; response bodies unclosed on non-200; connections checked out before the raise.
5. Per retry: what proves idempotency; attempt cap AND total-elapsed cap inside the caller's own timeout (`requests` needs the `(connect, read)` tuple — e.g. `timeout=(3.05, 10)` — never unbounded); confirm no other layer retries the same call; confirm attempt 2 sees the same input (no consumed stream/popped queue — retry a from-scratch closure over immutable inputs).
6. Kill-test each multi-step effect: simulate process death between write 1 and write 2; restart must converge — no stranded state, no double effect.
7. Chain intact end-to-end: `raise ... from err` / `%w` at every type translation; each message adds only its own layer's discriminating values ("user 4211", "after 3 attempts") without restating the child; logged exactly once at the handling layer; secrets/PII redacted at construction time, not log time.
8. The boundary handler's own reporting is wrapped in a last-ditch primitive write (stderr): the logger throwing on an exotic attached object masks the original error — this is the one place a bare catch is right.
9. C/errno-style: capture the error code immediately after the failing call — any intervening call shadows it.
10. Go write paths: check `Close()`'s error explicitly (flush failure at close = data loss reported as success); bare `defer f.Close()` is fine only for reads. Don't `%w`-wrap internal sentinels you don't want in your public API — `errors.Is`-reachable means contract (Hyrum's law applies to error chains and error strings alike).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 12 claims: 11 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found: Opus still recommends Java checked exceptions "for recoverable conditions" (outdated doctrine — the skill's main correction); it omits the library-vs-application swallowing-policy rule; otherwise it matched or exceeded the old skill (it cited RFC 9457 unprompted, which the skill hadn't).
- Consequence: skill restructured from a survey into a correction sheet + applied-review checklist; the checklist exists for the knowing-vs-doing gap, not knowledge transfer.
