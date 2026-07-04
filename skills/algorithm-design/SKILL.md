---
name: algorithm-design
description: Load when designing an algorithm from scratch — competitive-programming problems, interview questions, or production code where the naive approach is too slow. Covers problem classification, invariant-first design, greedy proof obligations, reductions to known problems, two-pointer/sliding-window/monotonic-stack triggers, randomization, and systematic boundary handling.
---

# Algorithm Design

## Core mental model

- **Classify before you design.** Every problem is one of: search (find any/all valid X), optimization (find the best X), counting (how many X), or decision (does X exist). The class picks the technique family: decision problems often admit binary search on the answer; counting problems almost never admit greedy; optimization problems with "choose a subset/sequence" smell like DP or exchange-argument greedy.
- **The expert's first move is reduction, not invention.** Before designing anything, ask: is this sorting in disguise? Shortest path on an implicit graph? Bipartite matching? Max-flow/min-cut? Interval scheduling? An expert spends the first two minutes trying to *recognize*, not create. A novel algorithm is a last resort and a red flag — most contest/interview problems are a known problem wearing a costume.
- **Design the invariant first; the algorithm writes itself.** State precisely what is true after each step ("after processing i elements, `best[j]` holds the minimum cost using exactly j items", "the window [l, r] is always the maximal valid window ending at r"). If you can't state the invariant in one sentence, you don't have an algorithm yet — you have vibes. The invariant also *is* your correctness proof and your loop-body checklist.
- **Complexity budget from constraints.** n ≤ 20 → 2^n or n·2^n (bitmask). n ≤ 500 → n³. n ≤ 5000 → n². n ≤ 10⁵–10⁶ → n log n or n. n ≤ 10⁹+ → log n, O(1) math, or binary search on the answer. Compute the budget *before* designing; it prunes the technique space brutally and tells you when your current idea cannot possibly work.
- **Monotonicity is the master key.** Binary search, two pointers, sliding window, monotonic stack/queue, and greedy all rest on some monotone structure ("if a window of size k works, so does k−1"; "the answer as a function of the threshold is monotone"). When stuck, hunt for the monotone quantity.

## Decision framework: pattern triggers

| Signal in the problem | Technique | Why |
|---|---|---|
| "Minimum/maximum X such that condition holds" + condition is monotone in X | Binary search on the answer | Converts optimization → decision; you only need a checker |
| Contiguous subarray/substring + "longest/shortest satisfying P" where P is monotone under shrinking | Sliding window (two pointers) | Each pointer moves only forward: O(n) total |
| Sorted input (or sortable) + pair/triple with target relation | Two pointers from both ends | Monotone elimination: one comparison kills one candidate |
| "Next greater/smaller element", "largest rectangle", "visible items", stock-span | Monotonic stack | Each element pushed/popped once; the stack stores the frontier that could still matter |
| Sliding-window max/min | Monotonic deque | Same idea over a moving window |
| "Choose non-overlapping intervals / deadlines / minimize lateness" | Greedy by earliest end time (prove via exchange) | Classic exchange argument territory |
| Order statistics, "k-th smallest", median | Quickselect (expected O(n)) or heap of size k | Don't full-sort for one statistic |
| Independent subproblems that combine cheaply (merge, count crossings) | Divide and conquer | See recognition signals below |
| Count inversions / closest pair / "pairs with property across halves" | D&C with a combine step | The combine step is the whole design problem |
| Equal-probability sampling from a stream, hashing adversarial input, "with high probability" | Randomization | See below |
| Constraints ≤ 20 items, "subsets", "orderings" | Bitmask enumeration / bitmask DP | 2^n fits the budget |

Additional high-frequency triggers:

| Signal | Technique | Why |
|---|---|---|
| Many range-sum queries, no updates | Prefix sums | `sum(l, r) = P[r] - P[l]` with half-open convention |
| Many range *updates*, one final read | Difference array | Add v on [l, r): `d[l] += v; d[r] -= v`; prefix-sum once at the end |
| 2-D region sums | 2-D prefix sums | Inclusion-exclusion: `P[r2][c2] - P[r1][c2] - P[r2][c1] + P[r1][c1]` — the sign pattern is where bugs live |
| "Count subarrays with sum k" (negatives allowed) | Prefix sum + hash map of counts | Window fails on negatives; `count += seen[P[i] - k]` |
| n ≈ 40, subsets, "exact/target" | Meet in the middle | Split into halves: 2·2²⁰ enumerations + sort/hash join beats 2⁴⁰ |
| "At most k" AND "at least k" variants | Solve "at most", subtract: exactly(k) = atMost(k) − atMost(k−1) | Direct "exactly" is usually much harder |
| Cyclic array problems | Concatenate (conceptually) a+a, or handle wraparound case separately | E.g., max circular subarray = max(normal Kadane, total − min subarray) |

**Divide-and-conquer recognition signals:** (1) the problem on the whole array decomposes into left half + right half + *interactions across the midpoint*, and the interaction can be handled in O(n) — inversions, closest pair, maximum subarray; (2) the recursion depth is log n and work per level is linear; (3) merge sort is already computing your answer as a side effect (inversions = swaps merge sort would do). If the "combine" step needs the full sub-solutions rather than a summary, D&C won't beat brute force.

**Reduction checklist (run it every time):**
- Can I sort and does order then make the problem trivial or greedy-able?
- Is there a graph here? (States = nodes, moves = edges → BFS/Dijkstra. See graph-algorithms skill.)
- Is "assign X to Y with constraints" bipartite matching or flow?
- Is "minimum removal to disconnect/separate" a min-cut?
- Is the problem a known NP-hard one (subset-sum with big values, TSP, set cover)? Then stop optimizing for exact polynomial and go for: small-n exact (bitmask DP), pseudo-polynomial DP, or an approximation/heuristic — and *say so* rather than shipping a wrong greedy.

## Greedy: proof obligations, not vibes

A greedy without a proof sketch is a bug generator — greedy is the technique with the highest ratio of "feels right" to "is right."

- **Exchange argument (the workhorse):** Assume an optimal solution OPT differs from greedy's choice at the first decision point. Show you can swap OPT's choice for greedy's choice without making OPT worse. Conclude greedy's first choice is safe; induct. If you cannot execute this swap argument in your head in ~30 seconds, treat the greedy as *conjecture* and test it against brute force on small inputs before trusting it.
- **Classic sorting-key greedies and their exchange proofs:** interval scheduling → earliest finish (swapping in the earlier-finishing interval never blocks more); minimize weighted completion time → sort by p_i/w_i (adjacent swap changes cost by w_j·p_i − w_i·p_j); "form largest number from pieces" → sort by comparator `a+b > b+a` (adjacent exchange). Note the pattern: prove an *adjacent* swap never helps, which validates the sort order.
- **Known greedy traps:** coin change is greedy-safe for canonical systems (US coins) but *not* in general — {1, 3, 4} making 6: greedy gives 4+1+1, optimal is 3+3. 0/1 knapsack by value density is wrong (fractional knapsack is the greedy-safe variant). "Take the locally cheapest edge/step" fails whenever choices interact through a shared budget.
- **Matroid smell (optional but decisive):** if the feasible sets are closed under subsets and satisfy the exchange property (any smaller independent set can grow from a larger one), greedy-by-weight is provably optimal. Spanning trees (Kruskal) and "pick k items with distinct labels" are matroids; general knapsack is not.

## Randomization as a design tool

- **Use it when:** an adversary could craft worst-case input (randomized quicksort/quickselect pivots, random hash seeds), you need a canonical fingerprint (Rabin-Karp rolling hash, Zobrist hashing for game states), sampling from a stream (reservoir sampling: keep item i with probability k/i), or verification is cheap but construction is hard (try random candidates + check).
- **Calibration:** expected O(n) quickselect is usually better in practice than the deterministic median-of-medians O(n) (huge constant). Rolling-hash equality needs collision awareness: with a 64-bit modulus and random base, collisions are negligible for contest sizes, but a *fixed* base/modulus can be attacked — randomize the base at runtime.
- **Las Vegas vs Monte Carlo:** prefer Las Vegas (always correct, random runtime) when you can verify; only accept Monte Carlo (probably correct) when you can quantify and bound the error probability.

## Failure modes & pitfalls

- **Shipping greedy without the exchange argument.** Correction: write the one-paragraph exchange sketch, or brute-force-check n ≤ 8 inputs first. If you find yourself writing "intuitively, taking the largest first..." — stop, that's the bug being born.
- **Binary search on a non-monotone predicate.** Before binary-searching the answer, explicitly verify: if check(x) is true, is check(x±1) true in the right direction? Predicates like "exists a partition into exactly k parts" are often *not* monotone in k even when they feel like it.
- **Binary search boundary bugs.** Standardize on ONE template and never improvise: `lo, hi = 0, n` (hi exclusive); `while lo < hi: mid = (lo + hi) // 2; if ok(mid): hi = mid else: lo = mid + 1`; answer is `lo`. This finds the first true in a false…false-true…true array. The three classic bugs: `while lo <= hi` mixed with exclusive updates (infinite loop), `mid = (lo+hi)//2` with `lo = mid` (infinite loop when hi = lo+1 — must use `mid = (lo+hi+1)//2` when the surviving side is lo), and forgetting that "no true exists" leaves `lo == n`, which you must check before indexing.
- **Sliding window on a non-shrink-monotone predicate.** The window trick requires: if [l, r] is valid then [l+1, r] is valid (or the mirrored version). "Subarray with sum exactly k" with negative numbers breaks this — use prefix-sum + hash map instead. Check for negative numbers before reaching for the window.
- **Two pointers that skip valid pairs.** The elimination argument must hold: when you advance a pointer, prove no future answer needed the abandoned element. In 3-sum-style problems, the sorted-order argument does this; in ad-hoc problems, capable engineers advance the wrong pointer and silently lose answers.
- **Monotonic stack direction confusion.** "Next greater element" → pop while stack top ≤ current (maintain strictly decreasing stack). Popping on `<` vs `≤` decides how ties resolve, and ties are where these solutions go wrong — decide tie semantics *from the problem statement* (e.g., largest rectangle in histogram needs one side strict and one non-strict to avoid double-counting or missing equal-height bars).
- **The off-by-one class, handled systematically instead of by prayer:**
  - Fix conventions once per solution and write them as a comment: intervals are half-open `[l, r)`; `prefix[i]` = sum of first i elements (so `sum(l, r) = prefix[r] - prefix[l]`, no −1 anywhere).
  - Half-open intervals compose: length is `r - l`, adjacent intervals share an endpoint, empty interval is `l == r`. Closed intervals generate +1/−1 corrections at every seam — avoid them unless the problem's data is inherently inclusive.
  - Test mentally on n = 0, n = 1, and n = 2 *before* running. Most boundary bugs are visible at n = 1.
  - When a loop processes "pairs of adjacent elements", the loop bound is `n - 1` items — say out loud which index is the *left* of the pair.
- **Integer overflow in mid/products.** In Python irrelevant, but in C++/Java: `mid = lo + (hi - lo) / 2`, and cast before multiplying (`(long long)a * b`). If writing Python as a spec for another language, flag these.
- **Binary search on reals with an epsilon exit.** `while hi - lo > 1e-9` can loop forever (float granularity near large values) or exit too early. Correction: iterate a fixed count — 100 halvings shrink any initial interval below any meaningful epsilon; `for _ in range(100)` is unconditionally safe and simpler to reason about.
- **Amortized analysis missed → wrong complexity claim.** A loop containing a while-loop is not automatically O(n²): if the inner loop consumes a resource each element produces at most once (stack pops, pointer advances), total work is O(n). Conversely, don't claim O(n) for a two-pointer where a pointer can move *backward* — re-check the "only forward" property.
- **Modular arithmetic slips.** Take the modulus after every add/multiply, not once at the end (other languages overflow; Python bigints get slow). Subtraction: `(a - b) % MOD` is already non-negative in Python but negative in C++/Java — add MOD before taking `%` when speccing for those. Division needs the modular inverse (`pow(b, MOD-2, MOD)` for prime MOD), never `//`.
- **Floating-point equality in geometry/greedy comparisons.** Sorting by an angle or ratio and comparing with `==` misgroups equal keys. Compare cross-products/cross-multiplied fractions in integers whenever inputs are integral (`a1*b2 vs a2*b1`), reserving floats for output only.
- **Optimizing before classifying.** Micro-optimizing an O(n²) that the budget says must be O(n log n) wastes the whole session. Re-derive the budget from constraints first; if current complexity exceeds it, the *approach* is wrong, not the constants.

## When stuck: the unstick sequence

Run these in order; each takes under a minute.
1. Re-read the constraints; recompute the budget. A budget of n log n with no sort in sight means binary search on the answer or a heap/window.
2. Sort the input (by each plausible key) and stare — order reveals greedy and two-pointer structure.
3. Ask "what is the last decision?" (unlocks DP) and "what is the state graph?" (unlocks BFS/Dijkstra).
4. Solve the problem with one constraint deleted, then reintroduce it — the delta names the technique.
5. Solve n = 1, 2, 3 by hand and diff the answers — patterns in the deltas suggest recurrences or closed forms.
6. Invert: instead of building the answer, count/remove the complement ("min removals to satisfy P" = n − "max kept satisfying P" — the kept version is often a classic).

## Worked micro-examples

**1. Binary search on the answer (minimize max load).** Split array `a` into k contiguous parts minimizing the maximum part sum. Classify: optimization → decision ("can we split with max sum ≤ x?"). The checker is greedy (pack until exceeding x, then cut) and the predicate is monotone in x.

```python
def split_array(a, k):
    def parts_needed(cap):          # greedy checker: fewest parts with each sum <= cap
        cnt, cur = 1, 0
        for v in a:
            if cur + v > cap:
                cnt, cur = cnt + 1, v
            else:
                cur += v
        return cnt
    lo, hi = max(a), sum(a)         # lower bound: must fit largest element
    while lo < hi:
        mid = (lo + hi) // 2
        if parts_needed(mid) <= k:  # monotone: bigger cap never needs more parts
            hi = mid
        else:
            lo = mid + 1
    return lo
```
Invariant: `lo` ≤ answer ≤ `hi` at every step; the greedy checker is itself proven by an exchange argument (cutting as late as possible never increases part count).

**2. Exchange argument executed (job with deadlines and profits, unit time).** Jobs (profit, deadline); schedule ≤1 per slot to maximize profit. Greedy: sort by profit descending, place each job in the latest free slot ≤ its deadline. Proof sketch: suppose OPT skips the highest-profit job J that greedy placed. Adding J to OPT requires evicting some job J' from J's slot (or an empty slot — immediate win). Since profit(J) ≥ profit(J'), the swap doesn't decrease OPT. Induct on decisions. The "latest free slot" choice keeps earlier slots open — a second, separate exchange shows placing later never hurts. Two obligations, two sketches; only then write code (with union-find for "latest free slot" if n is large).

**3. Monotonic stack (daily temperatures / next greater).**
```python
def next_greater(a):
    res, stack = [-1] * len(a), []      # stack holds indices with strictly decreasing values
    for i, v in enumerate(a):
        while stack and a[stack[-1]] < v:   # '<' : equal values wait for a strictly greater one
            res[stack.pop()] = i
        stack.append(i)
    return res
```
Invariant: the stack always contains exactly the indices whose next-greater is not yet found, in decreasing value order. Each index is pushed and popped at most once → O(n). The tie decision (`<` not `<=`) came from the spec "strictly greater."

## Verification / self-check

- State the invariant in one sentence. Walk it through: initialization (true before loop?), maintenance (each iteration preserves it?), termination (invariant + exit condition ⇒ answer correct?).
- Confirm complexity against the constraint budget with actual numbers (10⁵ · log(10⁵) ≈ 1.7M — fine; (10⁵)² = 10¹⁰ — dead).
- Adversarial micro-tests before declaring done: empty input, single element, all-equal elements (tie handling), sorted and reverse-sorted, negative numbers if the domain allows them, and the extreme constraint values.
- For any greedy or pruning step: either cite the exchange/monotonicity argument or brute-force cross-check on all inputs of size ≤ 8 (`itertools` makes this a 5-line harness — write it).
- Re-read the problem statement once after solving; the most common expert-level failure is solving a slightly different problem (contiguous vs not, at-most vs exactly, strictly vs non-strictly).
