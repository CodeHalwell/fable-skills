---
name: code-review-mastery
description: Load when reviewing a diff, pull request, or patch — or when asked to find bugs in proposed changes before merge. Covers incident-causing bug patterns, reviewing for absent code (missing invalidation/rollback/timeout), severity calibration, and avoiding nitpick noise.
---

# Code Review Mastery

A strong cold review already covers the standard canon (error-path focus, retry idempotency, half-open time ranges, severity tiers, mock footguns). This sheet keeps only the habits and checklist rows that cold reviews still miss.

## The habits cold reviews skip

- **Open the code the diff doesn't show.** For every changed function, its callers; for every changed shared state, its *other writers* — grep the table/key/variable. Findings like "the revoke path doesn't invalidate this new cache" come from that grep, not from the diff. Reviewing hunk-by-hunk in the diff viewer is the root failure mode: the bug lives in the interaction with unchanged code.
- **Generate the obligation checklist before reading line-by-line.** List what the change *obligates* (table below), then verify each obligation is discharged — in this diff or demonstrably already handled. Reacting only to visible lines misses the incident-causing class entirely.
- **Ask the concurrency context of every touched function, not whether the diff mentions concurrency.** A module-level cache dict added to code running in a threaded server, a non-atomic lazy-init, check-then-act on the filesystem — none of these diffs say "thread" anywhere.
- **Budget comments; per-line scrutiny collapses after ~400 lines.** Review the riskiest files first (state, concurrency, money, auth) while fresh; request a split when refactoring is mixed with behavior change — the behavior change hides in the refactor noise. Every nitpick depletes the author's attention for your one critical finding.

## Obligation checklist (diff does X → verify Y exists)

| The diff... | Verify |
|---|---|
| Adds caching | Invalidation on *every* writer of the underlying data (grep them); TTL backstop — process-local caches never see each other's invalidations; size bound; **key includes every input that affects the value** (locale, user, version, flag state — the commonly-missed one) |
| Adds a network/DB/IPC call | Explicit timeout (many clients default to none); down-dependency behavior decided on purpose (fail open vs closed) |
| Adds a retry | Idempotency; backoff + jitter; retry cap |
| Multi-step write | Transaction or per-step compensation; the step-2-fails-after-step-1-committed story |
| Adds a queue/listener/subscription | **Backpressure or bound; dead-letter/poison-message handling; unsubscribe/cleanup on shutdown** — the row most often skipped |
| Adds config/feature flag | Missing-config behavior; flag off = old behavior, verified |
| Changes schema/data format | Dual-read/migration; previous binary can read the new writes (rollback story) |
| Adds a new query pattern | Supporting index — instant on dev's 100 rows, an outage on prod's 100M |
| Adds user-facing input | Size/length limits; pathological-case rejection (10MB name, 0-length file, negative quantity) |
| Deletes/deprecates code | All callers actually gone, including reflection/string-dispatch/config references |
| Adds a lock | Ordering vs existing locks; nothing slow held under it; released on exception paths |
| Adds a background job/cron | Overlap protection; an alert when it silently stops |

Meta-absences: tests for the *failure* cases this diff creates; the metric/log line that reveals prod breakage before users do; rate-limiting/auth on any new endpoint.

## Test-review corrections

- **Copied-then-edited tests are the top source of wrong assertions** — the pasted assertion still checks the *original* scenario. When test bodies are near-duplicates, diff them against each other.
- **Async tests missing `await` pass instantly** — the coroutine never runs, nothing is asserted.
- The one question that catches tautologies: **"does this test fail on main?"** For a bugfix PR, mentally revert the fix; if the new test still passes, it tests nothing.
- Behavior changed but no test changed → the behavior is untested, the tests are tautological, or the change is a no-op. All three are findings.

## Severity discipline

- Blocking = reachable correctness/data-loss/security bug, resource leak, unbounded growth, missing request-path timeout. Should-fix = errors too poorly logged to debug an incident, missing failure-mode test, O(n²) on user-controlled n, unit-ambiguous APIs (`timeout` → `timeout_seconds`). Everything else = "nit:"/"optional:", never blocks; order comments by severity.
- More than ~5 nits means the real feedback is "this needs a design conversation," not 20 comments.
- Two calibration failures to check for in *yourself*: de-escalating real risk because the author is senior or the deadline is near; and approving a correct-looking fix for the wrong root cause — especially "added a null check" fixes that mask an upstream invariant violation.

## Pre-submit check

1. Did I open the callers and the shared state's other writers, or only read the diff?
2. Did the obligation checklist run against everything this diff adds?
3. Would each new test fail on main?
4. Comments severity-ordered, nits marked optional?
5. One sentence: what this change does and the riskiest way it fails in production. Can't say it → I read it, I didn't review it.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 11 claims: 8 baseline (cut/compressed), 3 partial (sharpened), 0 delta.
- Opus's cold review is near-expert: it independently produced the Yuan et al. OSDI'14 error-handling stat, the half-open `BETWEEN` fix, the `assert_called_wiht` footgun plus autospec/`mock.seal()` defenses, payment-retry timeout-ambiguity reasoning, and a *fuller* review of the cache-PR example than the skill's own worked example (both cut).
- Remaining gaps this sheet centers: queue/listener obligations (backpressure, dead-letter, unsubscribe), cache-key completeness (locale/user/version/flags), copied-then-edited and missing-`await` test patterns, and the explicit open-callers/other-writers grep habit.
