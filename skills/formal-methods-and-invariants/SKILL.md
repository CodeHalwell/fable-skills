---
name: formal-methods-and-invariants
description: Load when reasoning about code correctness beyond "tests pass" — designing loop/class/protocol invariants, writing concurrent or distributed code, modeling protocols as state machines, choosing property-based tests, placing assertions, or proving termination. Also load when reviewing code where a subtle correctness bug is suspected.
---

# Formal Methods & Invariants

## Core mental model

1. **The invariant is the program.** Every correct loop, class, and protocol is organized around a property that is (a) established initially, (b) preserved by every step, (c) strong enough at exit/quiescence to imply the goal. Write the invariant *before* the code. If you can't state it, you don't yet understand the problem — the code you'd write is guessing.
2. **Correctness = induction.** "Invariant holds at start" is the base case; "every transition preserves it" is the inductive step. When you claim code is correct, you are implicitly claiming this induction goes through. Make the claim explicit and check each transition — especially the ones you didn't write (exceptions, early returns, reentrancy, concurrent interleavings).
3. **Contracts localize blame.** Precondition = caller's obligation; postcondition = callee's promise; invariant = everyone's promise. A bug is always a broken contract *somewhere specific*. When debugging, binary-search over contracts, not over lines.
4. **Small-scope hypothesis.** Most design bugs manifest in tiny instances (2 threads, 3 states, lists of length ≤ 3, one message reordering). Exhaustively checking a small scope (model checking, property-based tests with small sizes, hand enumeration) finds bugs that random large-scale testing misses, because bugs are dense in small state spaces and sparse in random traces.
5. **Concurrency correctness is never obvious.** Any claim of "this is clearly fine" about concurrent code without an interleaving argument is unfounded. The number of interleavings of two threads with n and m steps is C(n+m, n) — for n=m=10 that's 184,756. Intuition samples a handful.
6. **Termination needs a variant.** For every loop/recursion, name a quantity in a well-founded order (usually a non-negative integer) that strictly decreases each iteration. No variant, no termination argument — "it obviously finishes" has shipped infinite loops.

## Decision frameworks

**Which invariant kind do I need?**

| Situation | Tool | What to state |
|---|---|---|
| Writing any nontrivial loop | Loop invariant | Property true before each iteration test; at exit, `invariant ∧ ¬guard ⇒ goal` |
| Class with interdependent fields | Class (representation) invariant | Relation among fields true between public method calls; check every method preserves it, including error paths |
| Protocol / lifecycle (connection, order, job) | State machine | Explicit states, allowed transitions, per-state invariants; reject all other transitions |
| Concurrent shared state | Global invariant + atomicity argument | Which invariant each critical section preserves; what holds at every possible preemption point |
| Distributed system | Safety invariant + liveness property | Safety: "never two leaders", checkable per-state. Liveness: "eventually elected", needs fairness assumptions |
| Money/inventory/resources | Conservation invariant | Sum over all accounts/pools is constant (or changes only via named operations) |

**Verification method selection:**
- Pure function with rich input space → **property-based testing** (Hypothesis in Python, proptest in Rust, fast-check in TS). Costs minutes, finds boundary bugs unit tests miss.
- Concurrent algorithm, distributed protocol, or state machine with ≤ millions of reachable states → **model-check mentally or with TLA+/TLC**. Enumerate states at small scope (2–3 nodes/threads). This is the *only* reliable way to validate a novel lock-free or consensus-like design; testing cannot cover interleavings.
- Sequential code with tricky index arithmetic → **loop invariant + hand-verify 3 iterations + boundary cases (empty, singleton, full)**.
- Refactoring → **refinement**: show every observable behavior of the new code is a behavior of the old (or of the spec). Concretely: same postconditions under same preconditions; new code may strengthen postconditions or weaken preconditions, never the reverse (Liskov direction).

**Property-based testing — which property?** In descending power:
1. **Round-trip**: `decode(encode(x)) == x` (serializers, parsers, codecs).
2. **Oracle**: fast implementation agrees with slow-obvious one (`my_sort(xs) == sorted(xs)`).
3. **Invariant/metamorphic**: `len(result) == len(input)`, `f(x + y) == f(x) + f(y)`, inserting then deleting restores state.
4. **No-crash / type-correct** — weakest; still worth having for parsers of untrusted input.
Avoid re-implementing the function under test as the property — that just tests it against itself.

**Assertion placement strategy:** assert where an invariant is *supposed to have been restored*, not where it's used. Priority order: (1) class-invariant check at end of every mutating method, (2) postcondition of complex computations, (3) precondition at trust boundaries (public API, deserialized data), (4) "unreachable" branches (`assert False, f"unhandled state {s}"` in exhaustive dispatch). Don't assert what the type system already guarantees. In production, keep cheap asserts on (conservation, state-machine transitions); a crashed process is better than corrupted money.

## Failure modes & pitfalls

- **Writing the loop, then retro-fitting an invariant.** The invariant becomes a description of the (possibly wrong) code rather than a spec. Correction: state invariant → derive initialization (make it true trivially) → derive body (preserve it while progressing) → derive guard (exit when invariant + ¬guard gives the answer).
- **Invariant too weak to be inductive.** Classic: proving binary search correct with "target is in arr[lo:hi] *if present*" — you must also carry "arr is sorted" and precise half-open bounds, or the inductive step fails. If you can't prove the step, *strengthen* the invariant; don't weaken the claim.
- **Off-by-one from mixed interval conventions.** Half-open `[lo, hi)` composes: length is `hi-lo`, split point `mid` gives `[lo,mid)` and `[mid,hi)` with no overlap/gap. Closed intervals need `mid±1` adjustments that people get wrong. Standardize on half-open; when you see `while lo <= hi` with `hi = mid - 1`, check the closed-interval bookkeeping line by line.
- **Class invariant broken mid-method, then an exception escapes.** The object is now permanently corrupt but alive. Correction: mutate a local/copy then commit, or restore the invariant in `finally`, or poison the object (set a `_corrupt` flag checked by every method).
- **Check-then-act races.** `if key not in d: d[key] = make()` — two threads both pass the check. Any code shaped "read, decide, write" on shared state is broken without atomicity (lock held across *both*, or CAS loop, or `dict.setdefault`). This includes filesystem versions: `if not os.path.exists(p): open(p, 'w')` → use `open(p, 'x')`.
- **Believing a data structure's thread-safety covers your compound operation.** `queue.get()` then `queue.task_done()` are individually safe; your invariant spanning both is not. Atomicity of parts ≠ atomicity of the whole.
- **"I added a lock" without stating what the lock protects.** A lock is meaningless without an associated invariant: "L protects fields x, y and the invariant x == len(y)". Every access to x, y must hold L — check *reads* too; a racy read of a two-field invariant can observe the broken intermediate state.
- **Deadlock from unordered lock acquisition.** Transfer(a→b) locks a then b; Transfer(b→a) locks b then a. Correction: impose a global lock order (e.g., by account id: `first, second = sorted([a, b], key=id)`), and state it as a protocol invariant.
- **State machine with implicit states.** Code uses booleans `is_connected`, `is_closing`, `handshake_done` — 8 combinations, of which 3 are meaningful and 5 are reachable-by-bug. Correction: one enum, transitions only via a single `transition(from, to)` function that asserts legality. If you receive an event not valid in the current state, that's a *decision* (drop? error? queue?) — make it explicit per state, don't let it fall through.
- **Confusing safety and liveness.** "The system never processes a payment twice" (safety — violated by a finite trace) vs "every payment is eventually processed" (liveness — violated only by an infinite trace, needs fairness assumptions). Retries help liveness and *threaten* safety (duplicates); idempotency keys restore safety. If your fix for a timeout adds a retry, immediately ask what safety invariant the retry can now break.
- **Termination variant that doesn't strictly decrease on every path.** `while lo < hi: mid = (lo+hi)//2; ... lo = mid` — when `hi == lo+1`, `mid == lo`, no progress, infinite loop. The variant `hi - lo` fails to decrease on the `lo = mid` branch. Fix: `lo = mid + 1` on the branch that has excluded `mid`, and re-verify the invariant still holds with that exclusion.
- **Trusting "it passed 10,000 random tests" for concurrent code.** Schedulers are not adversarial; the bad interleaving may need a preemption in a 2-instruction window. Use small-scope enumeration, stress with more threads than cores + injected sleeps at suspicious points, or a model checker.
- **Property-based test with a generator that can't reach the bug.** E.g., generating dicts with string keys when the bug needs a key colliding hash; or floats without NaN/inf/-0.0. Inspect generated-value statistics (`hypothesis` `note`/statistics) and explicitly add adversarial cases to the strategy.
- **Refinement violation during "pure refactor".** New version returns items in different order, and one caller depended on order. Refinement check: enumerate observable outputs (return values, exceptions, side effects, *ordering*, timing-visible behavior) and confirm each is preserved or provably unobserved.

## Worked micro-examples

The two that carry non-obvious discipline (a strong model reproduces the mechanics cold; the *practice* is the point):

**1. Property test needs TWO properties to pin a spec.** For `merge_intervals`, a single structural property ("output sorted & disjoint") passes a merge that silently *drops* intervals; a single coverage property ("same points covered") passes one that returns an overlapping mess. You need both — one structural invariant plus one oracle/coverage check — or the test ratifies a broken implementation. The common failure is writing one plausible property and calling it tested. Bound the domain small (e.g. integers −50..50) so an exhaustive coverage oracle is feasible; that is the small-scope hypothesis paying rent.

**2. "Obviously atomic" is wrong in the majority of schedules.** `counter += 1` on two threads (L/A/S each): of the 20 program-order-preserving interleavings, only 2 give the correct final value 2 — 18/20 lose an update. The lesson is not the count (derivable) but the reflex: *decompose any shared-state operation into atomic steps and enumerate before asserting "clearly fine."* At 2 threads × 3 steps it is hand-tractable; that tractability is why small-scope enumeration beats stress-running.

## Verification / self-check

Before presenting correctness-sensitive code or analysis, confirm:
1. **Stated invariant?** Can you write, in one sentence each, the invariant of every loop and every class you touched? If not, go back.
2. **Induction checked?** Base case (initialization) verified concretely; every transition — including exceptions, early returns, and the *last* iteration — preserves it.
3. **Boundaries executed by hand:** empty input, single element, all-equal, all-negative/zero, max-size-1 and max-size. For intervals/indices: does the code handle `lo == hi`?
4. **Concurrent code:** identified every preemption point; enumerated interleavings at 2 threads, or explained the atomicity mechanism (which lock, which CAS) protecting each compound operation; stated the lock order.
5. **Termination:** named the decreasing variant and checked it strictly decreases on *every* branch, including the branch where the guard nearly holds.
6. **State machines:** every (state, event) pair has defined behavior — count them: |states| × |events|; unhandled pairs are bugs, not "won't happen".
7. **If any check above was skipped, say so explicitly** in the answer ("not verified for concurrent callers") rather than implying full correctness.
