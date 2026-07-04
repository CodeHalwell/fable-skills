---
name: code-review-mastery
description: Load when reviewing a diff, pull request, or patch — or when asked to find bugs in proposed changes before merge. Covers incident-causing bug patterns, reviewing for absent code (missing invalidation/rollback/timeout), severity calibration, and avoiding nitpick noise.
---

# Code Review Mastery

## Core mental model

- **Review the change's effect on the system, not the text of the diff.** The diff shows added/removed lines; the bugs live in the interaction between those lines and the unchanged code around them. Always open the surrounding function/file, the callers of changed functions, and the other writers of changed state. A diff that looks perfect in isolation can be a production incident in context.
- **The most dangerous bugs are in code that is ABSENT.** A diff cannot show the cache invalidation that wasn't written, the rollback that wasn't added, the timeout that wasn't set, the index that wasn't created. You must generate a checklist of "what this change *obligates*" and verify each obligation is discharged. Reviewers who only react to visible lines miss the incident-causing class entirely.
- **Error paths and concurrency cause incidents; happy paths cause bug tickets.** Studies of production failures repeatedly find the majority stem from incorrect *error handling* of *anticipated* errors — empty catch blocks, error paths that leak resources, retries without idempotency. Spend your review time where the incidents are: every `catch`/`except`/`errdefer`/`if err != nil`, every path where a lock, transaction, file, or connection is open when an exception can fly.
- **Your comment budget is finite.** Every nitpick you write depletes the author's attention for your one critical finding. Rank findings by expected production cost; deliver the top few forcefully and let style tools handle style.
- **Approval means "I would deploy this."** Not "I read it" or "looks reasonable." If you couldn't explain to an incident review why this change was safe, you haven't finished reviewing it.

## Where incident-grade bugs actually live (check these in order)

1. **Boundaries and off-by-ones on the changed edges.** Any `<` vs `<=`, `len(x)` vs `len(x)-1`, inclusive-vs-exclusive range, pagination cursor, time-window `[start, end)`. When a diff moves a boundary, mentally execute n=0, n=1, n=limit, n=limit+1. Watch specifically for: `range(len(x)-1)` that should be `range(len(x))`, slicing `[:-1]` copied from somewhere else, `BETWEEN` in SQL (inclusive both ends — double-counts midnight rows against a `[start, end)` caller).
2. **Error paths.** For each new failure point: What happens to in-flight state? Is the resource released on *every* exit (early return, exception, break)? Does the error propagate with enough context, or vanish (`except Exception: pass`, `catch (e) {}`, ignored `err`)? Does a partially-completed multi-step operation get rolled back or does it strand state?
3. **Resource leaks.** Anything acquired without `with`/`defer`/`try-finally`/RAII: file handles, DB connections *checked out of a pool*, locks, goroutines/tasks (a spawned task that never gets awaited/cancelled is a leak), event-listener registrations, temp files. Pool checkout without `finally: pool.release(conn)` is the classic slow-burn incident — works fine until the pool drains under load at 3 a.m.
4. **Concurrency introduced into "unrelated" code.** A diff needn't mention threads to create a race: adding a module-level cache dict to code that runs in a threaded web server; making a lazy-init non-atomic (`if _instance is None: _instance = build()`); moving a read outside a lock "for performance"; reusing a client/session object that isn't thread-safe (e.g., a shared `requests.Session` is fine; a shared DB cursor is not); check-then-act on the filesystem (`if not exists: create` → TOCTOU). Ask: "what is the concurrency context of every function this diff touches?" — not "does this diff mention concurrency?"
5. **State mutation and aliasing.** Function now mutates an argument the caller reuses; returns an internal list the caller modifies; default mutable argument (`def f(x, acc=[])`); a "copy" that's shallow where nested state is mutated.
6. **Unit/encoding/time bugs.** ms vs s (a timeout of `30` passed to an API expecting ms = instant timeout; expecting s = 8-hour hang), naive vs aware datetimes compared, local-time day boundaries, float money arithmetic, bytes vs str at I/O edges.
7. **Injection & trust boundaries on changed inputs.** New string interpolation into SQL/shell/HTML/log lines; new deserialization of external data; a validated field now sourced from a different, unvalidated place.

## Reviewing for what's ABSENT — the obligation checklist

When the diff does X, verify the paired obligation exists somewhere (in this diff or demonstrably already handled):

| The diff... | Obligation to verify |
|---|---|
| Writes to a cache / adds caching | Invalidation on every write path to the underlying data; TTL as backstop; key includes all inputs that affect the value (locale? user? version?) |
| Adds a network/DB/IPC call | Explicit timeout (most client libraries default to none or minutes); retry policy; behavior when the dependency is down — fail open or closed, decided on purpose |
| Adds a retry | Idempotency of the retried operation; backoff + jitter; retry budget/cap (unbounded retries turn one outage into a self-DDoS) |
| Multi-step write (two tables, DB + queue, API + local state) | Transaction, or explicit compensation/rollback for each partial-failure point; what happens if step 2 fails after step 1 committed? |
| Adds a queue/listener/subscription | Backpressure or bound; dead-letter/poison-message handling; unsubscribe/cleanup on shutdown |
| Adds config/feature flag | Behavior for *missing* config; safe default (flag off = old behavior, verified) |
| Changes a data format/schema | Reads of old-format data still work (migration or dual-read); rollback story — can the previous binary read the new writes? |
| Adds a new query pattern | Supporting index (a full scan that's instant on dev's 100 rows is an outage on prod's 100M) |
| Adds user-facing input | Length/size limits, rejection of the pathological case (10MB name, 0-length file, negative quantity) |
| Deletes/deprecates code | All callers actually gone (grep, including reflection/string-dispatch/config references); data written by old code still readable |
| Adds a lock | Consistent lock ordering with existing locks (deadlock); nothing slow/blocking held under it; released on exception path |
| Adds a background job/cron | Overlap protection (what if the previous run is still going?); observability when it silently stops |

Also ask the meta-absences: where are the tests for the *failure* cases this diff creates? Where's the metric/log line that will tell us this is broken in prod before users do? Where does this get rate-limited/authenticated if it's a new endpoint?

## Severity calibration

Tag every finding; order comments by severity; never let a Sev-3 comment appear above a Sev-1.

- **Blocking (must fix):** correctness bugs on any reachable path, data loss/corruption, security (injection, authz bypass, secrets in code/logs), resource leaks, missing timeout on a request-path dependency, unbounded growth (memory, queue, retries), backward-incompatible change without migration.
- **Should fix (strong push, can merge with follow-up only if truly urgent):** error messages/logging too poor to debug an incident, missing test for a failure mode this diff introduces, performance cliff at plausible scale (O(n²) where n is user-controlled), confusing API that callers *will* misuse (bool positional params, unit-ambiguous names like `timeout` — say `timeout_seconds`).
- **Consider (author's call, no re-review needed):** naming, structure, simplification, style beyond the linter. Prefix explicitly: "nit:" or "optional:". If you have more than ~5 of these, your real feedback is "this needs a design conversation," not 20 comments.
- Calibration errors to avoid: escalating taste to blocking ("I'd have used a different pattern" is not a defect); *de*-escalating real risk because the author is senior or the deadline is near; blocking on hypothetical future requirements ("what if we someday need multi-region?") when the code is correct for stated requirements.

## Reviewing tests as first-class code

- **A test that cannot fail is worse than no test** — it's a false safety signal. Look for: assertions on the mock instead of the behavior (`mock.assert_called_once()` as the *only* assert), `assertTrue(result)` where result is a non-empty list (always truthy), try/except around the assertion, async tests missing `await` (pass instantly), snapshot tests blindly regenerated in the same PR.
- **Check that the test would have failed before the fix.** For a bugfix PR, mentally (or actually) revert the fix and run the new test. If it still passes, it tests nothing. Ask the author: "does this test fail on main?"
- **Tests copied-then-edited** are the top source of tests that assert the wrong thing — the copied assertion still checks the *original* scenario. When you see near-duplicate test bodies, diff them carefully.
- **Test-only diffs deserve real review:** shared fixture mutations (breaks other tests via shared state), sleeps as synchronization (flake factory), `time.now()` without freezing (fails at month-end/midnight/DST), tests asserting on ordering that isn't guaranteed (dict/set/query without ORDER BY).
- If the diff changes behavior and no test changed, one of three things is true: the behavior isn't tested (ask for a test), the tests are tautological, or the change is a no-op. All three are findings.

## Failure modes and pitfalls (reviewer-side)

- **Line-by-line myopia.** Reviewing hunk-by-hunk in the diff viewer without opening callers. Correction: for every changed function signature or behavior, list the call sites and check at least the non-obvious ones. For every changed shared variable, find its other readers/writers.
- **LGTM-by-fatigue on large PRs.** Scrutiny per line collapses after ~400 lines. Correction: for big PRs, review the riskiest files first (state, concurrency, money, auth) while fresh; explicitly request a split if the PR mixes refactoring with behavior change — the behavior change hides in the refactoring noise.
- **Trusting the PR description.** "Simple refactor, no behavior change" primes you to skim. Verify the claim: a true refactor has bit-identical behavior — look for changed constants, reordered operations with side effects, `==` becoming `is`, changed exception types (callers catching the old type now miss it).
- **Reviewing the code the author wrote instead of the problem they solved.** Ask: does this diff actually fix the linked issue for all its cases? A correct-looking fix for the wrong root cause is the subtlest approval failure — especially "added a null check" fixes that mask an upstream invariant violation.
- **Nitpick displacement.** Ten style comments and zero on the unguarded `except: pass` in the same file. Style comments feel productive because they're easy to generate. Force yourself to answer "what breaks in production?" before writing any comment.
- **Assuming the tooling caught it.** Type checkers don't catch ms-vs-s, wrong-but-well-typed logic, missing invalidation, or race conditions. CI green means "the tested paths pass," not "correct."
- **Not executing the code mentally with hostile values.** For each changed function, spend 30 seconds on: empty input, huge input, duplicate entries, None/null, negative, concurrent double-invocation, and the same call replayed twice (retry semantics).
- **Missing the security implication of a "convenience" change.** Logging a whole request object (now the auth token is in logs), widening a query for debugging (now IDOR), catching-and-continuing on signature verification failure.

## Worked micro-example: the absent-code incident in a clean-looking diff

```python
# PR title: "Cache user permissions to cut DB load"
+PERMS_CACHE: dict[int, set[str]] = {}
+
 def get_permissions(user_id: int) -> set[str]:
-    return db.fetch_permissions(user_id)
+    if user_id not in PERMS_CACHE:
+        PERMS_CACHE[user_id] = db.fetch_permissions(user_id)
+    return PERMS_CACHE[user_id]
```

Every visible line is correct. The blocking findings are all absent code:
1. **No invalidation.** `revoke_permission()` elsewhere writes the DB but not this cache → revoked users keep access until process restart. That's a security incident, not a staleness nit. (Find it by grepping for other writers of the permissions table.)
2. **Unbounded growth.** One entry per user forever → memory leak shaped like a slow OOM weeks later. Needs an LRU bound (`functools.lru_cache(maxsize=...)` or TTL cache).
3. **No TTL backstop** even with invalidation — multi-process deployments won't see each other's invalidations; process-local cache of security data needs a short TTL (and a stated tolerance: "revocation may take up to 60s").
4. **Race (minor here, pattern matters):** check-then-act on the dict is atomic enough under the GIL for CPython, but the double-fetch under concurrent misses means a thundering herd on a hot key after restart — worth a comment, not a block.

Correct review output: one blocking comment (invalidation + TTL, with the grep evidence of the revoke path), one blocking (bound the cache), one "should" (document staleness tolerance), zero nits.

## Verification / self-check before submitting a review

1. Did I open the changed functions' **callers** and the shared state's **other writers**, or did I only read the diff?
2. For each new failure point: can I say what happens to resources, partial state, and the caller? If I can't, that's a comment.
3. Did I run the **obligation checklist** (invalidation/rollback/timeout/index/migration/bound) for what this diff adds?
4. Did I check the **tests would fail without the change**, and that no assertion is tautological?
5. Are my comments **severity-ordered**, with blockers unambiguous and nits marked optional — and would I personally deploy this if the blockers were fixed?
6. Can I state in one sentence what this change does and the riskiest way it could fail in production? If not, I haven't reviewed it — I've read it.
