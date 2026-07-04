---
name: testing-strategy
description: Load when designing tests, writing a test suite, deciding what/how to test, choosing between unit/integration/e2e, diagnosing flaky tests, or evaluating test quality. Covers property-based testing, boundary selection, mocking pitfalls, mutation-testing mindset, and what not to test.
---

# Testing Strategy

## Core mental model

- **A test's value is P(catches a real future bug) × cost-of-that-bug − maintenance cost.** Most suites are full of tests with near-zero P(catch) — they re-assert what the code obviously does — and missing tests where bugs live: boundaries, error paths, concurrency, integration seams. Optimize bug-catch per line of test code, not coverage percentage.
- **Test behavior through the public contract, not implementation.** A test that breaks when you refactor (behavior unchanged) is negative-value: it taxes every future change while catching nothing. If renaming a private method or reordering internal calls breaks tests, those tests assert implementation.
- **The mutation-testing question is the quality bar:** "if I inserted a plausible bug here (flip `<` to `<=`, drop a `not`, return early, off-by-one the loop), which test fails?" If the answer is "none," coverage is theater. Apply this mentally to every function you test; run a real mutation tool (`mutmut`, `cosmic-ray`, `Stryker`, `pitest`) on critical modules.
- **Bugs cluster at boundaries and in error paths**, not in the middle of the happy path. Effort allocation should look nothing like the code's line distribution.
- **Determinism is non-negotiable.** A test that fails 1% of the time will be retried, then ignored, then it will mask a real failure. Every source of nondeterminism (time, randomness, ordering, network, shared state) must be controlled at the test boundary, not tolerated.

## Choosing test level: pyramid economics, honestly

| Level | Catches | Misses | Cost driver |
|---|---|---|---|
| Unit (pure logic, no I/O) | Logic, boundaries, algorithms | Wiring, config, contract mismatches | Cheap to write/run; brittle if over-mocked |
| Integration (real DB/queue via testcontainers, real module wiring) | Serialization, SQL correctness, transactions, contract mismatches between your modules | Cross-service issues, infra config | Setup complexity; seconds not ms |
| E2E | Deployment/config/wiring across services | Precise localization of failures | Flake, minutes, shared-env contention |

Rules that beat "70/20/10" dogma:
- **Push each bug class to the cheapest level that can catch it** — but not cheaper. SQL correctness cannot be unit-tested against a mock; testing it there yields a mock-shaped test that passes forever. Use a real database in a container. Conversely, don't test parsing logic through HTTP e2e — extract and unit test it.
- If your "unit" tests mock 4+ collaborators, the design has too much coupling OR the test belongs one level up. Mock count is a design smell meter.
- Keep e2e to a handful of **journey smoke tests** (signup→purchase→refund). Every e2e test you add must pay rent: it should catch a class of failure no lower level can (deploy config, service discovery, auth wiring).
- Integration tests with real infra (testcontainers-python, embedded Postgres, real Redis) have quietly become cheap. When you're choosing between "mock the DB" and "5 extra seconds of container startup amortized over the suite," choose the real DB.

## Boundary-value selection (mechanical procedure)

For every input domain, test: the minimum, minimum−1 (expect rejection), maximum, maximum+1, zero, one, "many," empty, and the type's natural hazards. Concretely:
- **Collections:** `[]`, 1 element, 2 (smallest "many" — catches wrong-loop-var bugs that n=1 hides), duplicates, all-identical, pre-sorted and reverse-sorted (for anything order-sensitive).
- **Integers:** 0, 1, −1, boundary±1 for every documented limit, and the overflow edge if the language has one.
- **Strings:** `""`, single char, whitespace-only, embedded newline, non-ASCII (`"café"`, an emoji — catches len-in-bytes vs chars), very long, string that looks like a number, string containing the delimiter/quote your code splits on.
- **Time:** midnight, month-end (Jan 31 + 1 month?), Feb 29, DST transitions, epoch 0, timezone-aware vs naive mix, end-of-day inclusive/exclusive.
- **Floats:** never `assertEqual` on computed floats — use `math.isclose`/`pytest.approx`; test NaN behavior explicitly if inputs can be NaN (NaN != NaN silently falsifies comparisons).
- **Pairs/ranges:** start==end, start>end, adjacent ranges (overlap detection off-by-one lives here).

Equivalence-partition first (one representative per behavior class) so the boundary cases are additions, not a combinatorial explosion. If two inputs take the same code path to the same decision, testing both buys nothing.

## Property-based testing: where it dominates

Use Hypothesis (Python) / fast-check (JS) / proptest (Rust) when you can state any of these properties — each is a one-liner that replaces dozens of examples:
- **Round-trip:** `decode(encode(x)) == x` — serializers, parsers, codecs. The single highest-value property; finds encoding edge cases (surrogates, empty, nesting depth) no human enumerates.
- **Oracle:** fast/clever implementation matches slow/obvious one: `my_sort(xs) == sorted(xs)`.
- **Invariant:** output always satisfies P: sorted-order, balanced tree, non-negative balance, `len(merge(a,b)) == len(a)+len(b)`.
- **Metamorphic:** relation between calls without knowing either answer: `search(q, filters) ⊆ search(q)`; `f(x*2)` vs `f(x)` scaling; idempotence `normalize(normalize(x)) == normalize(x)`.

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_roundtrip(xs):
    assert decode(encode(xs)) == xs
```

Practices that separate experts: always **register the shrunk failing example** as a permanent `@example(...)` regression; constrain strategies to *valid* domain values or you'll test your validator instead of your logic; set `deadline=None` for slow code rather than letting Hypothesis flag slow examples as flaky; use `st.builds()` on your real constructors so generated objects satisfy real invariants. Don't use PBT where properties reduce to re-implementing the function in the test — that's the oracle pattern without an oracle, and it just duplicates bugs.

## When mocks lie

A mock encodes *your belief* about a dependency. The suite then verifies your code against your beliefs — bugs live exactly where belief diverges from reality:
- **Mocked errors that can't happen / missing errors that do:** you mock `requests.get` to raise `ConnectionError`, but the real failure mode is a 200 with an HTML error page from a captive portal, or a `ReadTimeout` mid-body. Mock the *actual* failure taxonomy of the dependency (read its docs/source), not the generic one.
- **Mocks that accept anything:** `Mock()` happily returns a `Mock` from any attribute — your code calls a method that doesn't exist and the test passes. In Python, always `Mock(spec=RealClass)` / `create_autospec(real_func)`; without `spec`, a typo'd assertion method like `mock.assert_called_wiht(...)` is itself silently swallowed as a no-op attribute access (this exact bug has shipped everywhere; modern mock versions catch `assert_*` typos but not arbitrary method drift).
- **Interaction tests frozen to implementation:** `mock.assert_called_once_with(exact, args)` breaks on harmless refactors and passes when the *result* is wrong. Assert outcomes (return value, state change, message enqueued) over interactions, except where the interaction IS the contract (e.g., "charges the card exactly once").
- **The contract-drift problem:** service A's tests mock service B; B changes; A's tests stay green; prod breaks. Countermeasures in order of cost: shared contract tests (Pact-style consumer-driven), integration tests against B's real test instance, or at minimum generating mocks from B's published schema (OpenAPI/protobuf) so drift fails the build.
- Rule of thumb: **mock things you own at seams you designed; fake things you don't own with high-fidelity fakes** (in-memory repo implementing the same interface + the same contract test suite run against both real and fake).

## Flaky-test root causes (diagnose by symptom)

| Symptom | Likely cause | Fix |
|---|---|---|
| Fails at specific times (midnight, month-end, DST, ~11:59) | Unfrozen `now()`; test data with relative dates crossing a boundary | Freeze time (`freezegun`, injected clock); never `sleep` to "wait for" time |
| Fails only in full-suite runs, passes alone | Shared state: module globals, class attrs, DB rows, env vars, un-reset singleton, leaked mock patch | Randomize order permanently (`pytest -p randomly`) to surface these early; isolate via fixtures with teardown; fresh DB schema/transaction-rollback per test |
| Fails only alone, passes in suite | Test depends on a previous test's setup | Same fix; this one is already lying about coverage |
| Fails under parallelism | Fixed ports, shared temp paths, same DB rows | Ephemeral ports (bind port 0), `tmp_path` fixture, per-worker DB/schema |
| Timeout-flaky in CI, fine locally | Real network calls; sleeps as synchronization; CI has fewer cores changing interleavings | Block real network in unit tests (`pytest-socket`); replace every `sleep(x)` with poll-until-condition-with-deadline or event/latch |
| Fails ~1/N with different values | Unseeded randomness; iteration over sets/dicts where order leaks into assertions; DB query without ORDER BY | Seed and log the seed; compare as sets/sorted; add ORDER BY when order is asserted |

Policy: a flaky test gets fixed or deleted within days — quarantine-with-ticket at most. Auto-retry-on-fail as a *permanent* mechanism trains the suite to hide real races (retries are acceptable only as a detection mechanism that files the flake).

## What NOT to test

- Language/stdlib/framework behavior (that getters get, that Django saves a model, that `json.dumps` works).
- Trivial code with no branches (dataclass field assignment, pure delegation) — mutation testing will show these tests kill no mutants.
- Private functions directly — test through the public API; if a private function is complex enough to demand direct tests, extract it into a public unit.
- Exact copies of the implementation (`assert tax(100) == 100 * RATE` where the test imports `RATE` from the code — recompute expected values *by hand* as literals: `assert tax(100) == 8.25`).
- Logging/metrics calls, except the few where the log line is the product (audit trails, billing events).
- Exhaustive combinations when pairwise/partition coverage exercises every code path — combinatorial suites cost run time and maintenance while adding no new path coverage.
- UI pixel/snapshot assertions on everything: broad snapshots fail on every change, get regenerated ritually, and thereby test nothing. Snapshot narrowly and only stable structures.

## Table-driven tests and fixture design

```python
import pytest

@pytest.mark.parametrize("desc,principal,days,expected", [
    ("zero principal",      0,      30,  0.00),
    ("one day",             1000.0,  1,  0.14),   # hand-computed, not derived from the code
    ("regular month",       1000.0, 30,  4.11),
    ("leap-year boundary",  1000.0, 366, 50.14),
], ids=lambda v: v if isinstance(v, str) else None)
def test_interest(desc, principal, days, expected):
    assert interest(principal, days) == pytest.approx(expected, abs=0.01)
```
- Every row needs an **id/description** — a failure that prints `test_interest[3]` wastes the first five minutes of debugging.
- Rows must be **behaviorally distinct** (different partition or boundary), not bulk variations of one case.
- The moment a row needs its own special-cased logic inside the test body (`if desc == "leap": ...`), split it into a separate test — conditional test bodies hide which path actually ran.

Fixtures: build **one obviously-valid baseline object** per domain type via factory (`factory_boy` or a plain `make_user(**overrides)` function), and have each test override only the field it's about — `make_user(email="")` tells the reader exactly what matters. Never share mutable fixtures across tests (function scope by default; wider scopes only for immutable/expensive things like containers). A 40-line setup block means the code under test has too many dependencies — that's design feedback, not a fixture problem. Deep-frozen "golden" fixture files shared by 200 tests become unchangeable — prefer builders.

## Verification / self-check

1. **Watch each new test fail** (revert the fix, or sabotage the code deliberately). A test you've never seen fail is unverified. For a bugfix: the test must fail on pre-fix code.
2. Run the mental mutation pass on the code under test: flip each comparison, drop each branch — name the test that dies. No test dies → add one or consciously accept the gap.
3. Run the new tests 10x in randomized order and in parallel (`pytest -p randomly -n auto --count=10` with pytest-repeat) before merging anything touching time, ordering, or shared state.
4. Grep your new tests for lies: `sleep(`, unseeded `random`, `now()`/`today()`, `assertTrue(` on a non-boolean, bare `Mock()` without spec, assertions inside `try/except`.
5. Check the expected values were computed independently of the implementation (by hand, by oracle, by spec) — not by running the code and pasting its output (that only enshrines current behavior; acceptable *only* for explicit characterization tests of legacy code).
