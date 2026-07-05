---
name: refactoring-safely
description: Load when restructuring existing code without changing behavior — renames, extractions, module reorganization, migrating legacy code, replacing a dependency, or evolving an API/schema in place. Covers characterization tests, seams, strangler-fig, parallel-change (expand/migrate/contract), and keeping diffs reviewable.
---

# Refactoring Safely

You already know the standard discipline; the checklist below is completeness anchors only. The section after it is the part practice actually gets wrong.

## Core discipline (checklist — one-liners, details are standard)

- Refactor = bit-identical observable behavior: exception types, error text callers parse, ordering, log/metric names, perf class, serialization bytes all count. Never mix refactor + behavior change in one commit; bugs found mid-refactor get a `TODO-later.md` entry, not an inline fix.
- Legacy spec = current behavior, bugs included (Hyrum's law). Characterization tests before touching untested code: sentinel-assert, paste actual values in, chase weird inputs, coverage scoped to the blast radius. Name them `test_characterization_*`; at scale, golden-master via a snapshot lib (`syrupy`) or JSON files + `git diff`.
- If code can't be tested at all: make the *minimal, ugliest* seam change (default-preserving injected parameter, extract-and-override) without a net — and change nothing else in that unprotected step.
- `mock.patch` targets where the name is *looked up*, not defined; default `autospec=True`.
- Parallel change (expand/migrate/contract) for anything with independent consumers. DB rename: add column → dual-write/read-old → backfill → read-new → stop-write-old → drop; every step compatible with both N and N+1 running concurrently. Contract only on measured zero old-path usage.
- Mechanical refactors: tool-executed (IDE rename, `libcst`/OpenRewrite), one operation per commit however large, reviewer verifies the transform. Judgment refactors: small commits, line-by-line review. Never hand-execute what a tool can do; never mix the two kinds in one commit.
- Before any rename, enumerate non-code observers: serialization keys, reflection/`getattr` strings, config, persisted data, metric/alert names, external consumers.
- No refactoring on a red or flaky suite — stabilize first, budget it as part of the refactor.

## Where strong-baseline practice still goes wrong (the delta)

- **Step size is governed by distance-from-green, not by care.** The reflex when a step breaks more than expected is to push through until it compiles again — that's how big-bang messes start. The Mikado rule instead: if you're more than a few minutes from green, **revert**, write down the prerequisite you just discovered, do that prerequisite first, then retry the original step. Reverting feels like lost work; it's the fast path.
- **Seam placement grain.** Standard advice ranks seam *mechanisms* (object > link > monkeypatch) but is silent on *where*. Put seams at domain boundaries (`fetch_rate`, `send_notification`), not utility grain (patching `requests.get` at each call site). Utility-grain seams multiply across the codebase and all break on the first implementation swap; a domain seam survives it.
- **Strangler fig — three sequencing moves nobody defaults to:**
  1. Ship the router/facade routing **100% to old** as its own first step. It's pure infrastructure with zero behavioral risk and unlocks every later slice; coupling "introduce router" with "migrate first slice" doubles the risk of both.
  2. Migrate **highest-change-frequency slices first**, not easiest-first. Easiest-first feels productive but leaves daily development trapped in legacy; churn-first moves the team's everyday work onto the new system early — which is what keeps the migration funded past the 60% mark where these projects die.
  3. **Shadow-launch risky slices** before cutover: send traffic to both, serve old, diff responses offline (GitHub's `scientist` pattern or a hand-rolled comparator). Expect benign diffs (timestamps, ordering) and triage them *before* trusting the signal, or the comparison is noise.
- **"Dead" code needs a tombstone over a full business cycle.** Grep and static inspection miss reflection, cron, and external callers — but even a usage counter at zero for three weeks isn't proof. Month-end and quarter-end jobs run monthly/quarterly, and they are exactly the callers that look dead. Instrument, then wait out the longest business cycle that could exercise the path.
- **Contract-phase decay is a scheduling failure, not a knowledge failure.** Everyone knows dual paths rot; they still never get removed because removal is nobody's ticket. Mechanical fix: file the contraction ticket — with a date and an owner — in the same PR that ships the expand step.
- **Backslide ratchet, concrete Python mechanism:** during a signature migration, make CI treat `DeprecationWarning` as error *in migrated packages only* (pytest `filterwarnings` per-package). Scoped so unmigrated modules don't block, and cheaper than a custom lint/semgrep rule.
- **Reviewability mechanics:** move first, edit second, **two commits** — a move+edit commit destroys git rename detection and forces the reviewer to diff the file as new. State "move-only, no edits" in the PR and prove it with `git diff --color-moved=dimmed-zebra -w`. A judgment PR that must exceed ~400 lines needs a review map: which files are mechanical fallout vs. which ~40 lines need brains.

## Verification / self-check

1. Pure refactor done = **tests unchanged and green**. Had to edit a test? Either it asserted implementation (fix it in a separate preparatory commit) or you changed behavior (stop, reclassify the PR).
2. Diff the observable surface, not the code: exception types (grep upstream `except`/`catch`), serialized output byte-for-byte where feasible, log/metric names alerts depend on, exported public names.
3. Migrations: old-path counter at zero over a full business cycle before contracting; reconciliation diff at zero before switching system of record.
4. Can each commit be reverted independently, right now, without a data fix or coordinated deploy? If not, re-plan that step before shipping it.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed), 2 partial (sharpened), 0 outright delta.
- Opus cold reproduces essentially the entire discipline — expand/contract with N/N+1 schema compatibility, the characterization sentinel trick, the `mock.patch` lookup-site gotcha (plus `autospec=True`, which the skill lacked), move-only proof via `--color-moved`/blob SHAs, and a perf-regression taxonomy broader than the skill's. Skill restructured to a correction sheet accordingly.
- Biggest residual gaps: strangler-fig sequencing (router-at-100%-old as step one, churn-first slice ordering, shadow/scientist diffing with benign-diff triage); domain-grain vs utility-grain seam placement; Mikado revert-and-resubdivide step-size rule; full-business-cycle tombstone window for dead-code removal.
