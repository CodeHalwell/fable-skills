---
name: dynamic-programming
description: Load when a problem involves optimizing/counting over sequences, subsets, intervals, or trees where brute force is exponential — recognizing DP in disguise, designing states, the four-step protocol, space optimization, top-down vs bottom-up, and the classic iteration-direction bugs.
---

# Dynamic Programming

## Core mental model

- **State design is the whole game.** A DP state is *the minimal summary of the past that determines the future*. Everything else — transitions, code, complexity — falls out of the state. When a DP is hard, you haven't found the right state; when a DP is wrong, the state is missing information the future actually needs. Spend 80% of design time asking: "given this summary, can I make all remaining decisions correctly without knowing anything else about how I got here?"
- **DP = brute-force recursion + a cache, no more.** Any correct exponential recursion whose argument space is polynomial becomes a polynomial algorithm mechanically. If you can't write the brute force, you can't write the DP — write the brute force first.
- **Two preconditions, both mandatory:** *optimal substructure* (optimal answer composes from optimal sub-answers — breaks when subproblems interact, e.g., longest *simple* path in a general graph) and *overlapping subproblems* (the recursion revisits states — otherwise plain divide-and-conquer suffices, no cache needed).
- **Count states × transition cost = complexity.** Before coding: |state space| × (work per state). If that exceeds the budget (see constraints: n ≤ 20 → bitmask 2^n·n; n ≤ 500 → n³ interval DP; n ≤ 10⁵ → n log n or n·k with small k), the state is too rich — find what to drop or aggregate.
- **"Order of processing" is a modeling choice.** Many DPs only work because you impose an order that kills symmetry: process items one at a time (knapsack), sweep left to right (LIS), shrink from both ends (interval), process a tree bottom-up. Choosing the decision order IS choosing the state.

## The four-step protocol (run it explicitly, in order)

1. **State.** Define `dp[...]` in one precise sentence including what's *decided* and what's *pending*. Bad: "dp[i] = best answer for i". Good: "dp[i][j] = minimum cost to convert the first i chars of A into the first j chars of B." If the sentence needs "somehow" or "roughly", stop and redo.
2. **Transition.** Enumerate the *last decision* that produced this state (usually easier than the first): `dp[i][j] = min over last-moves of (cost + smaller states)`. Verify every referenced state is strictly "smaller" under some well-founded order — this is what guarantees termination and dictates iteration order later.
3. **Base cases.** The states with no pending decisions. Get the *identity elements* right: counting DPs start at 1 ("one empty way"), min-DPs at 0/∞, max at 0/−∞. Wrong base cases are the #1 source of answers that are off by exactly one or exactly double.
4. **Order / memoization.** Bottom-up: iterate so all dependencies are computed before use. Top-down: `functools.lru_cache` and the recursion order handles it. Then read off where the answer lives (often NOT `dp[n][n]` — e.g., LIS answer is `max(dp)`, not `dp[n-1]`).

## Recognizing DP in disguise — family trigger table

| Trigger phrases / shape | Family | State skeleton |
|---|---|---|
| "minimum edits/operations to transform", "align two sequences", diff, spell-check | Edit distance / two-sequence | `dp[i][j]` over prefixes of both |
| "longest common ...", "interleaving", "is A a subsequence-merge of B, C" | Two/three-sequence | `dp[i][j]` prefixes |
| "select items with capacity/budget", "can we reach sum S", "min coins" | Knapsack (0/1, unbounded, bounded) | `dp[capacity]` or `dp[i][capacity]` |
| "count ways to make change / climb stairs / tile" | Counting knapsack/linear | same, with `+` instead of `min` |
| "burst balloons", "matrix chain", "merge stones", "remove boxes", polygon triangulation, "last operation on a range" | Interval DP | `dp[l][r]`, iterate by increasing length, transition over split point or *last* element removed |
| "count numbers ≤ N whose digits satisfy P" | Digit DP | `dp[pos][tight][state-of-P]`, walk digits of N |
| n ≤ 20 with "visit all", "assignment", "order matters with pairwise costs" | Bitmask DP | `dp[mask][last]` (TSP-style) or `dp[mask]` |
| "on a tree: max independent set / distances / subtree choices" | Tree DP | `dp[v][took_v?]`, combine children post-order |
| "rerooting: answer for every node as root" | Tree DP, two passes | down-pass + up-pass with prefix/suffix child aggregates |
| "longest increasing ...", "chains", "boxes nest inside boxes" | LIS family | sort one dimension, LIS the other; O(n log n) via patience |
| "maximum subarray / must-take-contiguous" | Kadane | `dp[i]` = best ending at i |
| "game, both play optimally, who wins / best score difference" | Game DP (minimax over states) | state = position; value = best margin for player to move |
| "probability / expected value after k steps" | Probability DP | `dp[step][state]`, transitions weighted by probabilities |
| "string can be segmented into dictionary words" | 1-D over cut points | `dp[i]` = prefix of length i decomposable |

Anti-triggers (looks like DP, isn't): all transitions costs equal and state graph explicit → plain BFS; subproblems don't overlap → divide and conquer; greedy exchange argument holds → greedy (e.g., interval *scheduling* is greedy; interval *partitioning DP* is for weighted variants).

## Failure modes & pitfalls

- **State missing future-relevant information.** Symptom: small test cases pass, adversarial ones fail. Classic: "max product subarray" needs BOTH max and min ending at i (a negative flips them); "paint house" needs which color was used last; digit DP needs the `tight` flag or you count numbers exceeding N. Debug method: find the failing input, trace which two different histories collapsed into one state but needed different futures — that difference is the missing state component.
- **0/1 vs unbounded knapsack iteration direction (the single most common DP bug).** With 1-D `dp[c]`, iterating capacity **descending** means each item is used at most once (you read pre-item values); **ascending** means unlimited reuse (you read post-item values). Memorize as: *descending = each item frozen before it can feed itself*.
```python
for w, v in items:                       # 0/1: capacity DESCENDING
    for c in range(cap, w - 1, -1):
        dp[c] = max(dp[c], dp[c - w] + v)
for w, v in items:                       # unbounded: capacity ASCENDING
    for c in range(w, cap + 1):
        dp[c] = max(dp[c], dp[c - w] + v)
```
- **Counting: combinations vs permutations by loop nesting.** Items outer, capacity inner → each item considered once in a fixed order → counts *combinations* (coin change ways). Capacity outer, items inner → counts *ordered sequences* (climbing stairs with step sizes). Swapping these loops silently changes the question being answered — decide which the problem asks, then pick nesting deliberately.
- **Iteration order violating dependencies.** Bottom-up requires topological order of the dependency DAG. Interval DP must iterate by increasing `r - l` (not `for l: for r:` naively, which reads uncomputed `dp[l+1][r]` — actually that one works for some transitions and not others, which is why it's insidious). Rule: for each `dp[X] uses dp[Y]`, verify Y is computed strictly earlier under your loop order — check the extreme corner (first iteration) by hand. When in doubt, go top-down with memoization: recursion computes dependencies on demand and cannot get the order wrong.
- **Base case identity errors.** `dp[0] = 1` for counting ("empty way"), not 0. Min-DP initialized with `float('inf')` — then guard the final answer against inf (unreachable ⇒ report -1, don't return inf). Max-DP over possibly-all-negative values: initializing to 0 silently allows "take nothing" — check whether the problem permits an empty selection.
- **Answer read from the wrong cell.** LIS: `max(dp)` not `dp[-1]`. Kadane: track global max, not final `dp`. Interval DP on "merge entire array": `dp[0][n-1]`, but on "best sub-interval": max over all cells.
- **Space optimization destroying path reconstruction.** Rolling arrays (`dp[i][*]` → two rows, or one row with careful direction) cut O(n·m) memory to O(m) — but the full table IS the breadcrumb trail for reconstructing *which* choices were made. If the problem asks for the actual subsequence/edit script/item set: keep the full table (or store separate compact `parent` pointers, or use Hirschberg's divide-and-conquer for edit-distance-style reconstruction in O(m) space). Deciding to roll arrays before reading whether the output is a value or a path is a rewrite waiting to happen.
- **Top-down in Python at scale.** `lru_cache` recursion at n ≥ ~10⁴ depth hits the recursion limit and is ~3–5× slower than a loop. `sys.setrecursionlimit` helps until it segfaults. Rule of thumb: top-down to *derive and validate* the DP, convert to bottom-up when n is large or TLE looms. Also: `lru_cache(maxsize=None)` — the default 128 silently degrades to exponential re-computation.
- **Mutable default / shared-row initialization.** `dp = [[0]*m]*n` aliases one row n times — writes to `dp[0][j]` appear in every row. Use `[[0]*m for _ in range(n)]`. This bug produces DP tables that look "smeared" when printed — print the table when confused.
- **Memoizing on unhashable or over-rich keys.** Caching on a list argument crashes; caching on an index-pair *plus* an accumulator that's derivable from the indices wastes cache space and can hide the true state space size (turning O(n²) states into O(n²·V)). The accumulator being in the key when it shouldn't be = your state was wrong.
- **Assuming optimal substructure that isn't there.** Longest simple path, "max sum with no two chosen elements sharing any prime factor" — subproblem solutions constrain each other globally. Test: can two different optimal sub-solutions be freely swapped without violating feasibility? If not, DP over that decomposition is unsound.

## Top-down vs bottom-up calibration

- Top-down wins when: state space is sparse (only reachable states computed — digit DP, game DP where most positions never occur), transitions are complex, you're still exploring the design, or dependency order is nontrivial.
- Bottom-up wins when: full table needed anyway, n is large (no recursion overhead/limits), you want rolling-array space savings (top-down can't roll), or inner loops can be vectorized (numpy row ops can accelerate 100×).
- Default workflow: derive top-down (it mirrors the recurrence exactly), then mechanically convert: cache → table, recursive calls → reads, recursion order → loop order matching the well-founded order from step 2.

## Worked micro-examples

**1. Edit distance — protocol executed.**
State: `dp[i][j]` = min ops converting `a[:i]` → `b[:j]`. Transition by last op: replace/match (`dp[i-1][j-1] + (a[i-1] != b[j-1])`), delete from a (`dp[i-1][j] + 1`), insert into a (`dp[i][j-1] + 1`). Base: `dp[i][0] = i`, `dp[0][j] = j`. Order: row-major, all deps up-left.
```python
def edit_distance(a, b):
    m, n = len(a), len(b)
    dp = list(range(n + 1))                    # rolling: dp = previous row
    for i in range(1, m + 1):
        prev_diag, dp[0] = dp[0], i            # dp[0] = base case dp[i][0]
        for j in range(1, n + 1):
            prev_diag, dp[j] = dp[j], min(
                prev_diag + (a[i-1] != b[j-1]),   # dp[i-1][j-1]
                dp[j] + 1,                        # dp[i-1][j]  (not yet overwritten)
                dp[j-1] + 1,                      # dp[i][j-1]  (already this row)
            )
    return dp[n]
```
The `prev_diag` juggling is the rolling-array hazard made explicit: one cell of the old row must survive its own overwrite. Note: this version cannot output the edit script — that requires the full table.

**2. Bitmask DP (TSP-style, n ≤ 20).**
State: `dp[mask][last]` = min cost to visit exactly the set `mask`, currently at `last`. Transition by last move: came from some `prev` in `mask ^ {last}`.
```python
import itertools
def tsp(dist):                                  # dist[i][j], n <= 20
    n = len(dist)
    FULL = (1 << n) - 1
    INF = float('inf')
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0                                # start at city 0
    for mask in range(1 << n):
        for last in range(n):
            d = dp[mask][last]
            if d == INF: continue
            rem = FULL ^ mask
            while rem:                          # iterate unvisited cities via lowbit
                nxt = (rem & -rem).bit_length() - 1
                rem &= rem - 1
                nm = mask | (1 << nxt)
                if d + dist[last][nxt] < dp[nm][nxt]:
                    dp[nm][nxt] = d + dist[last][nxt]
    return min(dp[FULL][last] + dist[last][0] for last in range(1, n))
```
Order is automatically valid: transitions go from `mask` to a strict superset, and we iterate masks ascending (a superset is always a larger integer). Budget check: 2²⁰ × 20 × 20 ≈ 4·10⁸ — tight in Python; fine in C++; for Python, n ≤ 16.

**3. Mechanical brute-force → DP conversion (word break).**
Brute force: `can(i) = any(s[i:j] in words and can(j) for j in range(i+1, n+1))`, `can(n) = True`. Argument space: one int → cacheable.
```python
from functools import lru_cache
def word_break(s, words):
    words, n = set(words), len(s)
    maxw = max(map(len, words), default=0)      # prune: transition cost n → maxw
    @lru_cache(maxsize=None)
    def can(i):
        if i == n: return True
        return any(s[i:j] in words and can(j)
                   for j in range(i + 1, min(n, i + maxw) + 1))
    return can(0)
```
The conversion touched nothing but adding the cache and a pruning bound — that is the standard, and the *only*, transformation needed. If your "conversion to DP" is changing the logic, you're introducing bugs.

## Verification / self-check

- Re-state the state definition sentence and check the transition uses ONLY information in it — any peek at "how we got here" means the state is wrong.
- Compute |states| × transition cost, compare to budget with real numbers.
- Hand-simulate the table on a tiny input (n = 3) and compare against brute force; better, run an automated cross-check against the exponential recursion for all inputs of size ≤ 8–10.
- Probe the classic bug sites: an input where an item could be reused (catches 0/1-direction), an all-negative input (catches base cases), an input where the answer is empty/zero (catches identity), the exact boundary n = 0 and n = 1.
- If space-optimized: confirm the problem asks for a value only, and confirm the surviving row is read *before* being overwritten (the `prev_diag` class of bug).
- If counting: confirm combinations vs permutations against a hand-counted n = 3 case, and apply the modulus at every addition, not just at the end (overflow in other languages; performance in Python).
