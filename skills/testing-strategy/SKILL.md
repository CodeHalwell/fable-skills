---
name: testing-strategy
description: Load when designing tests, writing a test suite, deciding what/how to test, choosing between unit/integration/e2e, diagnosing flaky tests, or evaluating test quality. Covers property-based testing, boundary selection, mocking pitfalls, mutation-testing mindset, and what not to test.
---

# Testing Strategy

Baseline model knowledge here is strong (mutation mindset, pyramid economics, flaky diagnosis, parametrized-test hygiene, real-DB-over-mock, independent expected values). This sheet keeps the compact checklists plus the corrections the baseline gets wrong or fumbles.

## Corrections the reflex gets wrong

- **A `Mock()` without `spec` swallows your assertions too.** The known trap is un-specced mocks accepting any method call from the code under test. The less-known one: a typo'd assertion like `mock.assert_called_wiht(...)` is itself just an attribute access — it returns a child Mock and the test passes while asserting nothing. Modern `unittest.mock` catches `assert_*`-prefixed typos, but not arbitrary method drift. Always `create_autospec(real)` / `Mock(spec=RealClass)` — for the assertions' sake, not only the code's.
- **Mock the dependency's real failure taxonomy, not the generic one.** The reflex is to mock `requests.get` raising `ConnectionError`. Real HTTP failure modes that ship bugs: a 200 with an HTML error page (captive portal, misconfigured proxy), a `ReadTimeout` mid-body after headers arrived, truncated JSON. Read the dependency's actual error surface (docs/source/recorded traffic) before writing the error-path mocks; otherwise the error tests verify fictions.
- **A fake is only trustworthy if it passes the real thing's contract tests.** "Use high-fidelity fakes for things you don't own" is common advice; the missing half is the enforcement: write one contract test suite and run it against BOTH the real implementation and the in-memory fake in CI. A fake without this decays into a mock with extra steps. Corollary: mock things you own at seams you designed; fake things you don't own — with a verified fake.
- **PBT anti-pattern: a property that re-implements the function.** If the only property you can state is `assert f(x) == <same formula as f>`, you've built the oracle pattern without an independent oracle — the test duplicates the implementation's bugs. Either find a weaker true property (round-trip, invariant, metamorphic) or use hand-computed examples instead.
- **Make nondeterminism fail loudly by default, permanently:** `pytest -p randomly` (order randomization as standing policy, not a one-off debug tool) so state leakage surfaces the week it's introduced; `pytest-socket` to hard-block real network in unit tests; bind port 0 for ephemeral ports under parallelism. Before merging anything touching time/ordering/shared state: run the new tests 10x randomized and parallel (`pytest -p randomly -n auto` + pytest-repeat).
- **Fixture design is a coupling meter.** Build one obviously-valid baseline object per domain type via factory (`make_user(**overrides)` / factory_boy); each test overrides only the field it's about — `make_user(email="")` states what matters. A 40-line setup block is design feedback (too many dependencies), not a fixture problem. Deep-frozen golden fixture files shared by 200 tests become unchangeable — prefer builders. Function scope by default; wider scopes only for immutable/expensive resources (containers).
- **n=2 is the smallest "many."** A collection test with only 0 and 1 elements can't catch wrong-loop-variable or wrong-accumulator bugs; 2 elements can. Include duplicates, all-identical, and pre-/reverse-sorted for anything order-sensitive.

## Checklists (completeness, not novelty)

**Boundary values per input domain** — min, min−1, max, max+1, zero, one, many, empty, plus:
- Collections: `[]`, 1, 2, duplicates, sorted/reverse-sorted.
- Integers: 0, 1, −1, documented-limit±1, overflow edge.
- Strings: `""`, single char, whitespace-only, embedded newline, non-ASCII/emoji (bytes-vs-chars length), very long, looks-like-a-number, contains your delimiter/quote.
- Time: midnight, month-end arithmetic (Jan 31 + 1 month), Feb 29, DST both directions, epoch 0, aware/naive mix.
- Floats: `pytest.approx`/`math.isclose` only, explicit NaN behavior (NaN != NaN).
- Pairs/ranges: start==end, start>end, adjacent ranges, endpoint inclusivity.
Equivalence-partition first; boundaries are additions, not a cartesian product.

**Flaky-test symptom → cause:** time-of-day failures → unfrozen `now()` (freeze it); fails in suite only → shared state (randomize order, teardown fixtures, per-test transaction rollback); fails alone only → depends on a sibling's setup; fails in parallel → fixed ports/paths/rows (port 0, `tmp_path`, per-worker schema); CI-timeout-only → real network or `sleep`-as-sync (block sockets; poll-with-deadline); ~1/N with varying values → unseeded RNG, set/dict order, missing ORDER BY. Policy: fix or delete within days; permanent auto-retry trains the suite to hide races.

**Don't test:** stdlib/framework behavior; branchless glue; private functions directly (extract if complex); expected values imported from the implementation (hand-compute literals); logging/metrics unless the log IS the product; exhaustive combos past path coverage; broad UI snapshots.

**Test-level economics:** push each bug class to the cheapest level that catches it — but not cheaper. SQL/serialization/transactions need a real containerized DB (same engine and version as prod); parsing logic needs extraction and unit tests, not e2e. 4+ mocks = the test belongs a level up or the design is too coupled. E2e = a handful of journey smoke tests that each catch a failure class no lower level can.

## Verification / self-check

1. Watch each new test fail (sabotage or pre-fix code); for a bugfix the test must be red on the unpatched commit.
2. Mental mutation pass: flip each comparison, drop each branch — name the test that dies, or add one.
3. Grep new tests for lies: `sleep(`, unseeded `random`, `now()`/`today()`, `assertTrue(` on non-boolean, bare `Mock()` without spec, assertions inside `try/except`.
4. Register every Hypothesis-shrunk failure as a permanent `@example(...)`.
5. Confirm expected values came from outside the implementation (hand, spec, oracle) — pasted output is acceptable only for explicit, human-reviewed characterization tests of legacy code.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 9 baseline (cut/compressed), 2 partial (sharpened), 1 delta (expanded).
- Baseline already nails: mutation-testing mindset + tools, boundary lists, PBT patterns + Hypothesis settings, contract tests/autospec, flaky symptom→cause table, quarantine policy, parametrized hygiene, real-DB-for-SQL, mock-count-as-design-smell, watch-it-fail, independent expected values.
- Biggest gaps found: PBT property-that-reimplements-the-function anti-pattern (silent); swallowed `assert_called_wiht` typo trap and realistic HTTP failure taxonomy (missed); verified-fake pattern of running one contract suite against real and fake (missed); `pytest-socket`/permanent `-p randomly` tool anchors (vaguer).
