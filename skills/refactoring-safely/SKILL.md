---
name: refactoring-safely
description: Load when restructuring existing code without changing behavior — renames, extractions, module reorganization, migrating legacy code, replacing a dependency, or evolving an API/schema in place. Covers characterization tests, seams, strangler-fig, parallel-change (expand/migrate/contract), and keeping diffs reviewable.
---

# Refactoring Safely

## Core mental model

- **Refactoring means bit-identical observable behavior.** If outputs, side effects, error types, ordering guarantees, or performance class change, it's not a refactor — it's a behavior change wearing a refactor's PR title, and it must be reviewed and tested as one. Guard the word: mixed "refactor + improvement" commits are where regressions hide, because reviewers skim what's labeled refactor.
- **Never refactor and change behavior in the same commit.** The two-hats rule is mechanical: refactor commit (tests unchanged and green) → behavior commit (tests change too). This makes `git bisect` decisive and review tractable. When you spot a bug mid-refactor, note it, finish or revert the refactor, fix the bug separately.
- **Safety comes from small reversible steps, each of which compiles and passes tests** — not from care and cleverness within one big step. The expert's superpower is decomposing a scary migration into 15 boring steps, each shippable and revertible. If you're ever more than a few minutes from green, you've taken too big a step; prefer revert-and-resubdivide over pushing through ("the Mikado method": revert, write down the prerequisite you discovered, do that first).
- **Legacy code's real spec is its current behavior, bugs included.** Callers may depend on the bug (Hyrum's law). Before changing untested code, pin current behavior with characterization tests; decide *separately and explicitly* whether any weirdness you find is spec or bug.
- **The riskiest refactors look mechanical.** "Just a rename," "just moving a file" changes: serialization keys, reflection targets, DB migration ordering, import side-effect order, public API surface. The judgment step in every mechanical refactor is enumerating who observes the thing you're changing.

## Characterization tests before touching legacy code

Goal: a tripwire net, not a good test suite. You are recording what the code *does*, not what it *should* do — this is the one context where generating expected values by running the code is correct.

Procedure:
1. Identify the entry points you'll be preserving (the seam you'll refactor behind).
2. Feed them a spread of inputs — typical, boundary, and *weird* (you especially need the weird ones; that's where behavior you don't understand lives).
3. Assert whatever comes out: return values, DB writes, emitted messages, thrown exception types. Golden-master style is fine at scale: serialize outputs for hundreds of generated inputs to files, diff on every run (`pytest` + snapshot lib like `syrupy`, or plain JSON files + `git diff`).
4. Crank up strictness: run with coverage against the code you're about to change; uncovered branches in the change zone need inputs that reach them, or an explicit decision to let them break.
5. Mark them clearly (`test_characterization_*`) — they are scaffolding with a different maintenance contract; after the refactor stabilizes, replace the valuable ones with intent-revealing tests and delete the rest.

If you literally cannot get the code under test (hard-wired I/O, statics, constructors doing work), make the *minimal, ugliest possible* change to introduce a seam first — extract-and-override, parameter injection with a default preserving old behavior — and accept that this tiny step is done without a net. Minimize the unprotected surface; don't "clean up while you're in there."

## Seam identification

A seam is a place where you can alter behavior without editing the code that has it. Ranked by preference:
- **Parameter seam:** dependency already comes in as an argument — swap it. Cheapest; look for it first.
- **Constructor/injection seam:** add an optional parameter defaulting to current behavior (`def __init__(self, clock=time.time)`), so no caller changes. This default-preserving move is the workhorse of legacy rescue: zero-risk to add, immediately testable.
- **Extract-and-override:** pull the untestable bits (`socket`, `now()`, `random`) into a method; subclass in tests to override. Ugly, transitional, fine.
- **Module/link seams** (monkeypatching, import substitution): last resort — they couple tests to import paths and break under refactoring. Note `unittest.mock.patch` must target *where the name is looked up* (`patch("billing.invoice.fetch_rate")`, the importing module), not where it's defined — the classic silent-no-op patch bug.
- Choose seams at **domain boundaries** (fetch-rate, send-notification), not at utility grain (patching `requests.get` everywhere) — domain seams survive implementation swaps.

## Parallel change (expand / migrate / contract)

The universal pattern for changing anything with independent consumers — APIs, schemas, message formats, function signatures with many callers. Never force consumers to move atomically with the provider.

1. **Expand:** add the new thing alongside the old. Both work. (New column *nullable or defaulted*, new endpoint version, new parameter with compatible default, new field written *in addition to* old.)
2. **Migrate:** move consumers/data one at a time, each independently deployable and revertible. Dual-write during this phase; backfill old data; verify with a comparison job (count mismatches between old and new representations; drive to zero *measured*, not assumed).
3. **Contract:** only after telemetry shows zero old-path usage (log/metric on the old path — don't guess), remove the old thing. Schedule this or it never happens and you carry both paths forever, which is worse than never migrating.

Database-specific rules:
- Rename a column via expand/contract, never `ALTER ... RENAME` in place under a live app — old code is still running mid-deploy. Order: add new column → deploy code that writes both/reads old → backfill → deploy read-new → verify → drop old. Every step must be compatible with *both* the previous and next code version (N and N+1 run concurrently during rollout).
- `NOT NULL` on an existing column is a contract step: enforce in code first, backfill, then constrain.
- Adding an index or a `NOT NULL` with default can lock large tables depending on engine/version — check the specific DDL's locking behavior at your table size before running it in a migration that deploys block on.

API-specific: additive changes (new optional field) are free; anything else — removing a field, changing a type, changing semantics of an existing value — is expand/contract with a deprecation window and *usage telemetry* gating contraction.

## Strangler fig (replacing a system incrementally)

- Put an interception layer in front of the legacy system (router, facade, proxy). Route by feature/endpoint/tenant to old or new. Ship the router doing 100%-to-old first — that's a pure-infrastructure, low-risk step that unlocks everything.
- Migrate **one thin vertical slice at a time**, each carrying real traffic to the new system. Percentage rollouts + instant flag-flip rollback per slice.
- **Shadow/dark launch** before cutover on risky slices: send traffic to both, serve old, diff responses offline (GitHub's `scientist` pattern / a hand-rolled comparator). Expect and triage benign diffs (timestamps, ordering) before trusting the signal.
- The failure mode is the half-strangled system: 60% migrated, team disbands, both systems live forever. Countermeasures: migrate highest-change-frequency parts first (so daily work exits legacy early), keep a burn-down of remaining routes, and secure explicit commitment for the tail *before* starting.
- Data is the hard part, not code: decide the system of record per entity, one at a time; dual-write with reconciliation jobs; never let both systems be writable-authority for the same entity simultaneously.

## Mechanical vs. judgment refactors

| Class | Examples | Rules |
|---|---|---|
| Mechanical (tool-executed, whole-codebase) | IDE/LSP rename, move, inline, safe extract; `libcst`/OpenRewrite codemods | Do in one commit, however large; reviewers verify the *transform*, not each line ("codemod: X→Y, script attached"). Never hand-execute across many files what a tool can do — hand-execution injects typos at a rate per-file. |
| Judgment (semantic) | Changing abstractions, redistributing responsibility, replacing a pattern, de-duplicating "similar" code | Small commits, full review, tests first. Beware de-duplicating code that is *coincidentally* identical but serves different masters — merging it couples two change reasons; duplication is cheaper than the wrong abstraction. |

Watch for judgment hiding inside mechanical work: a rename that collides with an existing name in one file; a "move method" that changes which `self` state is captured; inlining a function that had a side effect callers relied on ordering of; regex-based "renames" that hit strings, comments, and serialized keys. Search non-code observers before any rename: config files, JSON/YAML keys, `getattr`/reflection strings, docs, DB values, log-based alerts.

## Keeping diffs reviewable

- **Stacked small PRs** over one 3,000-line PR: preparatory refactors first ("make the change easy, then make the easy change"). Each PR states its invariant: "no behavior change; tests untouched" or "behavior change X; see new tests."
- Do **format-only and move-only changes in isolated commits** so review diffs of the interesting commit are clean. Use `git diff --color-moved=dimmed-zebra` (and `-w`) to verify a move-only commit is truly move-only; put that claim in the PR description.
- A rename/move plus edits to the moved code in one commit destroys git's rename detection and the reviewer's ability to diff — move first, edit second, two commits.
- If a refactor PR must exceed ~400 lines of judgment changes, provide a review map: which files are mechanical fallout vs. which 40 lines need brains.

## Failure modes and pitfalls

- **"While I'm in here" scope creep** — the top refactoring killer. Each opportunistic fix widens the blast radius and un-mixes becomes impossible. Keep a `TODO-later.md`; finish the planned step.
- **Preserving your *understanding* of behavior instead of behavior.** You "simplify" a redundant-looking condition that actually guarded a rare case; you reorder two calls that "obviously" commute but the first mutates state the second reads; you replace exception type `ValueError` with a nicer custom error and break every `except ValueError` upstream. Exception types, error messages parsed by callers, and iteration/emission order are all observable behavior.
- **Refactoring on top of a broken or flaky test suite.** Red or flaky tests = no net; every "did I break that?" is unanswerable. Stabilize the suite first — it's part of the refactor's cost, budget for it.
- **Trusting green tests that don't cover the change zone.** Run coverage scoped to the files you're changing before starting; a 90% overall suite can be 0% on this module.
- **Big-bang branch migrations.** A months-long `refactor` branch diverges until merge is a rewrite. Refactor on main behind compatibility (parallel change), continuously integrated, or don't do it.
- **Skipping the contract phase.** Dual paths "temporarily" — the old column, the shim, the re-export — become permanent load-bearing confusion. File the contraction ticket with a date when you ship the expand step.
- **Removing "dead" code by static inspection alone.** Reflection, dynamic dispatch, cron configs, and external callers don't show in grep. Add a tombstone log/metric to "dead" code and let telemetry prove it dead over a full business cycle (month-end jobs run monthly).
- **Performance-class regressions inside behavior-preserving changes:** replacing a dict lookup in a loop with a `.filter()` per iteration; N+1 queries introduced by moving a fetch inside an extracted per-item method. Functional tests stay green; production melts. Eyeball complexity of hot paths; keep a benchmark for the truly hot ones.

## Worked micro-example: retiring a signature with 200 call sites

Goal: `send(user_id: int, msg: str)` → `send(recipient: Recipient, msg: Message)` across a live codebase. Atomic change = 200-file PR nobody can review. Parallel change instead:

```python
# Step 1 — EXPAND (one small PR): new path exists, old signature delegates.
def send(user_id: int, msg: str) -> None:
    warnings.warn("send(int, str) is deprecated; use send_v2", DeprecationWarning, stacklevel=2)
    send_v2(Recipient.from_user_id(user_id), Message(body=msg))

def send_v2(recipient: Recipient, msg: Message) -> None:
    ...  # real implementation moved here; old tests still pass unchanged
```
Step 2 — MIGRATE: codemod call sites in batches of ~20 (mechanical commits, one module per PR), CI treating `DeprecationWarning` as error *in migrated modules only* (pytest `filterwarnings` per-package) to prevent backsliding. Step 3 — CONTRACT (own PR, after grep + warning telemetry show zero callers): delete `send`, rename `send_v2` → `send` with an IDE rename in its own commit. Total: ~12 trivially reviewable PRs, each green, each revertible, main deployable throughout.

## Verification / self-check

1. **Tests unchanged and green** is the definition of done for a pure refactor — if you had to edit a test, either the test asserted implementation (fix it in a separate preparatory commit) or you changed behavior (stop, reclassify).
2. Diff the *observable surface*, not the code: same public names exported, same exception types raised (grep `except`/`catch` upstream for anything you changed), same serialized output byte-for-byte where feasible (golden-master diff), same log/metric names that alerts depend on.
3. For moves/renames: `git diff --color-moved -w` shows pure movement; anything not dimmed needs explanation.
4. For migrations: telemetry, not vibes — old-path counter at zero over a full business cycle before contracting; reconciliation diff at zero before switching system of record.
5. Ask: **can I revert each commit independently right now without coordination?** If any step's rollback requires a data fix or a synchronized deploy, re-plan that step before shipping it.
