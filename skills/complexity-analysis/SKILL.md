---
name: complexity-analysis
description: Loads when analyzing or predicting algorithm performance — deriving Big-O honestly (amortized/expected/worst-case), solving recurrences, judging whether a problem is NP-hard and what to do about it, explaining why the "slower" algorithm wins at real n (constants, cache), or auditing hidden complexity in library operations (list insert, string concat, dict resize).
---

# Asymptotic and Practical Complexity

## Core mental model

1. **Big-O is a family of claims — say which one you're making.** Worst-case, expected (over randomness *in the algorithm*: hash functions, pivots), and amortized (averaged over an operation *sequence*) answer different questions. `dict` lookup is O(1) expected but O(n) under adversarial collisions; `list.append` is O(1) amortized but O(n) on the resize event. Throughput questions want amortized/expected; per-operation latency SLOs and adversarial-input questions want worst case. Quoting the wrong one is how "O(1)" systems miss their p99.
2. **Asymptotics predict *scaling*; constants predict *time*.** O(n²) with a tight, cache-friendly inner loop beats O(n log n) with pointer chasing until n reaches the thousands. Production sorts use insertion sort below ~16–64 elements (Timsort, introsort cutoffs) for exactly this reason. The expert question is never "which is asymptotically better" but "where is the crossover, and which side of it is my n?"
3. **Memory access dominates modern constants.** L1 ≈ 1 ns; main memory ≈ 100 ns — a factor of ~100 hiding inside "O(1) access". Contiguous, predictable access (arrays, sort-then-scan) beats asymptotically equal linked/hashed structures at practical sizes; this is why "sort then group" often beats "hash then group", and why B-trees beat binary trees anywhere latency-per-hop matters.
4. **Lower bounds tell you when to stop optimizing the algorithm and change the problem.** Comparison sorting is Ω(n log n): distinguishing n! orderings needs log₂(n!) ≈ n log₂ n comparisons. Beating it requires *non-comparison* information — radix/counting sort on bounded integer keys runs O(n·k). Searching an unsorted array is Ω(n) by adversary argument: any uninspected cell could hold the target. At a lower bound, the remaining wins are constants — or restructuring so the bound's model no longer applies (index it, sort once, maintain an invariant).
5. **NP-hardness is a routing decision, not a death sentence.** Recognizing an NP-hard core redirects you from "find the efficient exact algorithm" (unavailable, as far as anyone knows) to the productive menu: exact exponential search that's fine at your n, ILP/CP-SAT solvers, approximation with guarantees, or local-search heuristics with an evaluation harness. Both failure directions are real: promising poly-time exactness for an NP-hard problem, and declaring a poly-time problem (assignment, max-flow, 2-SAT, shortest path) "intractable".
6. **Complexity composes multiplicatively through loops and calls.** The loop body's cost is whatever the *most expensive call inside* costs — including the innocent-looking library call. Most real quadratic blowups are one hidden O(n) operation inside an O(n) loop.

## Decision frameworks

### Recurrences
- **Master theorem** for T(n) = a·T(n/b) + f(n): compare f(n) against n^(log_b a). Leaf-dominated (f smaller by a polynomial factor) → Θ(n^(log_b a)); balanced → Θ(n^(log_b a) · log n); root-dominated (f larger + regularity condition) → Θ(f). Mechanically: mergesort a=2, b=2, f=n → n¹ = n¹ → Θ(n log n). Karatsuba a=3, b=2, f=n → n^1.585 dominates → Θ(n^1.585). Binary search a=1, b=2, f=1 → Θ(log n).
- **Where the master theorem does NOT apply**:
  - Unequal splits: T(n) = T(n/3) + T(2n/3) + n → recursion tree (below) → Θ(n log n).
  - Subtract-and-conquer: T(n) = T(n−1) + n → sum the series → Θ(n²); T(n) = 2T(n−1) + 1 → Θ(2ⁿ).
  - Non-polynomial gap: f = n log n vs n^(log_b a) = n falls between cases; answer for T = 2T(n/2) + n log n is Θ(n log² n), obtained from the tree, not the theorem.
  - Variable substitution cases: T(n) = T(√n) + 1 → set m = log n → Θ(log log n).
  - Randomized splits (quicksort): expectation argument over pivot positions, not the master theorem.
- Default fallback: **draw the recursion tree — per-level work × number of levels — then verify by substitution.** The tree never lies; the master theorem is three cached trees.
- **DP complexity = (#distinct states) × (work per state).** Count states honestly: bitmask over n items = 2ⁿ; interval DP = O(n²) states × O(n) split points = O(n³); (index, budget) = O(n·B) — which is *pseudo-polynomial*: B is exponential in its bit length, which is why knapsack's DP coexists with knapsack's NP-hardness.

### NP-hard recognition and the response menu
Smells: "choose a subset maximizing X under interacting constraints" (knapsack variants, set cover), "order/route through all items" (TSP, scheduling with precedence + deadlines), "partition into k groups minimizing cross-interactions" (coloring, correlation clustering), "satisfy all these boolean-ish rules" (SAT-like).
| Situation | Reach for |
|---|---|
| n ≤ ~20–25 | Exact exponential: bitmask DP (Held–Karp TSP: O(n²·2ⁿ)), branch-and-bound |
| n ≤ ~40, decomposable | Meet-in-the-middle (2^(n/2) enumeration) |
| Structured, up to ~10³–10⁵ variables, want (near-)optimal + proof | ILP (HiGHS/Gurobi) or CP-SAT (OR-Tools) — modern solvers routinely crush "NP-hard" instances with structure |
| Monotone submodular objective + cardinality constraint | Greedy: (1−1/e) ≈ 0.63 guarantee, near-best possible; usually the right answer to "pick k items" |
| Metric TSP-like | Nearest-neighbor construction + 2-opt/Or-opt improvement (Christofides for the guarantee) |
| Huge, time-boxed, no guarantee required | Local search / simulated annealing / LNS — validated against exact solves on shrunk instances |
Escape hatches to check *before* declaring hardness: tree/DAG/interval structure (many NP-hard problems turn polynomial there), a constraint that's actually always slack, 2-SAT sufficing instead of 3-SAT (poly), the "TSP" living on a line (sort), k fixed and tiny (n^k enumeration may be fine).

### Crossover estimation (constants and cache)
- If A costs c₁·n² (1 ns/op, contiguous) and B costs c₂·n log₂ n (20 ns/op, pointer-chasing), A wins while n < (c₂/c₁)·log₂ n ≈ 20 log₂ n → n ≲ 200. Measure c₁ and c₂ with a two-point microbenchmark; never guess them from the code's appearance.
- Cache misses and branch mispredictions contribute 10–100× to constants — the entire gap between "same Big-O" implementations. When two options tie asymptotically, pick the one that scans memory forward.
- Corollary: below n ≈ 100, almost everything is fast and the simplest code wins; asymptotic arguments start earning their keep around n ≈ 10⁴–10⁵ and become decisive by 10⁷.

### Space–time tradeoffs (state the exchange rate)
- Hash index: O(n) space buys O(1) probes replacing O(n) scans — break-even after ~2 lookups.
- Memoization: space = #states; time falls from tree size to DAG size (naive Fibonacci 2ⁿ → n).
- Prefix sums: O(n) space; range-sum queries O(n) → O(1). 2-D version for image/integral queries.
- Bloom filter: ~10 bits/key for ~1% false positives, zero false negatives — right when misses are cheap to double-check against ground truth.
- Recompute-vs-store (activation/gradient checkpointing): pay ~1.3–2× compute to cut memory to ~√-scale of layers.
- Always name what's bought, what's paid, and the break-even usage count; a tradeoff without its exchange rate is a slogan.

## Failure modes & pitfalls

- **Hidden quadratics in innocent library calls** — the highest-frequency real-world class:
  - `s += chunk` on strings in a loop: immutable → copy per iteration → O(total²). Use `''.join(parts)`. (CPython sometimes optimizes in-place concat; it disappears across refcounts and implementations — never rely on it.)
  - `list.insert(0, x)` / `list.pop(0)` in a loop: O(n) shift each → O(n²). Use `collections.deque` (O(1) at both ends).
  - `x in some_list` inside a loop: O(n) each → O(n²). Build a set once (O(n)), then O(1) membership.
  - `list.remove(v)`: O(n) search + O(n) shift — and mutating while iterating skips elements.
  - `np.append` / `pd.concat` in a loop: full reallocation each call → O(n²) plus memory churn. Accumulate in a Python list; one `np.concatenate`/`pd.concat` at the end.
  - Repeated `sorted(...)` to track top-k in a stream: use `heapq.nlargest` or maintain a bounded heap (O(n log k)).
  - `dict`/`set` growth: inserting n items triggers ~log-many rehashes, each O(current size) — amortized fine, latency-spiky; pre-size when the size is known.
- **Amortized ≠ smooth.** `list.append`'s occasional O(n) resize appears as rare big stalls that "O(1) amortized" hides from you but not from your p99. Pre-allocate (`[None]*n`) in latency-critical loops or accept spikes knowingly.
- **Slices and copies inside recursion.** `binary_search(arr[mid:])` turns O(log n) into O(n) per level; `sum(lst[:i])` inside a loop is the same bug flat. Pass indices, not slices; keep running aggregates.
- **Quoting O(1) hashing where the worst case is the contract** — untrusted keys can force collisions (hash flooding; Python randomizes string hashes for this reason) — and the reverse cargo-cult of avoiding dicts in offline analytics "because worst case".
- **Applying the master theorem where it's silent**: to T(n) = 2T(n−1) + 1 (subtractive → Θ(2ⁿ), not any master case), or to non-polynomial gaps. If subproblem size shrinks by subtraction, sum a series; if by root-taking, substitute variables.
- **"It's polynomial" via pseudo-polynomial DPs.** O(nW) knapsack is exponential in the *bit length* of W; with W = 10⁹ the "polynomial" DP allocates a 10⁹-wide table. Say "pseudo-polynomial" and check magnitudes before promising.
- **Misreading lower bounds as unconditional.** Ω(n log n) binds *comparison* sorts on arbitrary orderable keys — bounded integers admit O(n·k) radix sort legitimately. "Must read all input" Ω(n) dies when an index or sorted invariant already exists. Always name the model a bound lives in before invoking it.
- **Assuming the expected case without its randomness.** Quicksort's O(n log n) is expected over *pivot randomness*; first-element pivot on already-sorted input is Θ(n²) (introsort's heapsort fallback exists precisely for this). Hash-table O(1) assumes the hash spreads your actual keys — structured keys (strides, truncated timestamps) can cluster under weak hashes.
- **Optimizing the O while the constant lives elsewhere.** Replacing an O(n²) loop over n = 40 items (microseconds) while each iteration makes an O(1) network call (milliseconds) — the profile, not the asymptotics, ranks reality. Analyze to shortlist; measure to decide.
- **Ignoring output size.** "Enumerate all pairs/subsets matching X" is Ω(#answers); no algorithm beats its own output. If the answer set can be 10⁹ rows, change the question (count, sample, top-k, iterator), not the algorithm.
- **Big-O across layers.** An O(n) algorithm issuing n ORM queries is O(n) *round-trips* — the dominant term is in the units, not the count. State complexity in the resource that dominates (comparisons, cache lines, network calls, tokens).
- **Off-by-log sloppiness in interviews-turned-designs.** Sorting inside a loop over n items is O(n² log n), not O(n log n); binary search per element of an m-array over an n-array is O(m log n) and beaten by a hash or a merge when m ≈ n. Recompute the product every time the loop structure changes.

## Worked micro-examples

**1. String concat, quantified.** Building a 10⁶-char string from 10⁴ pieces of 100 chars: naive `+=` copies ≈ Σᵢ 100·i ≈ 100·(10⁴)²/2 = 5×10⁹ char-copies; `''.join` copies each char once ≈ 10⁶. A ~5000× gap from one idiom — this is the "report generator takes minutes" bug, and it reappears in every language with immutable strings (Java without StringBuilder, C# without StringBuilder, JS with naive `+=` in old engines).

**2. Recursion tree where the master theorem is silent.** T(n) = T(n/3) + T(2n/3) + cn. Each level's work sums to cn (the pieces partition n); the deepest path shrinks by 2/3 per step → depth log_{3/2} n ≈ 1.71 log₂ n → total Θ(n log n). Transferable insight: *any* fixed-fraction split gives n log n with only the constant changing — which is why quicksort survives mediocre pivots but dies on (1, n−1) splits, where the tree degenerates to depth n and Θ(n²).

**3. Amortized analysis of list growth, from first principles.** Doubling capacity on overflow: pushing n items costs n (writes) + 1 + 2 + 4 + ... + n (copies at resizes) ≤ n + 2n = 3n → O(1) amortized per push, by the aggregate method. Growing by a *fixed increment* k instead: copies cost k + 2k + ... + n ≈ n²/2k → O(n) amortized per push — growth must be geometric. Same analysis explains why shrinking should hysteresis (shrink at 1/4, not 1/2, occupancy) to avoid thrash at a boundary.

**4. NP-hard recognition redirect, end-to-end.** "Assign 200 tasks to 20 workers; each task has a qualified-worker subset; some task pairs conflict (can't share a worker); minimize max load." Conflicts + makespan = coloring/scheduling hybrid → NP-hard; do not promise an efficient exact algorithm. Route: CP-SAT model — booleans x[t,w] restricted to qualified pairs, `AddExactlyOne` per task, per conflict pair and worker at most one, minimize the max of worker loads — solves this size in seconds with optimality proof or gap. Fallback if a solver can't be a dependency: greedy (order tasks by fewest qualified workers, assign to least-loaded feasible worker) + swap/2-move local search, validated against CP-SAT on 30-task instances. Hardness spotted → solver-or-heuristic chosen → guarantee stated: that routing is the expert deliverable.

## Verification / self-check

- **Empirical scaling check**: time at n, 2n, 4n. Ratios ≈ 2 → linear; ≈ 4 → quadratic; ≈ 2 with slow creep → n log n; fit the exponent as log₂(t₂ₙ/tₙ) if unsure. If measurement disagrees with your analysis, a hidden copy/rehash/library call is missing from the analysis — believe the measurement, then find the term.
- **Recurrence check by substitution**: plug the claimed Θ back into the recurrence and confirm both sides balance; spot-check T(1), T(2), T(4) by hand.
- **Hot-loop library audit**: for every call inside the hot loop, state its true cost (`in` on list vs set, `insert(0)`, slicing, `+=` on str, DataFrame ops); the loop's complexity is (iteration count) × (worst call inside).
- **Label the case**: worst / expected / amortized — and confirm the label matches the caller's need (throughput vs p99 vs adversarial input).
- **For hardness claims**: name the known NP-hard problem that embeds into yours (reduction direction: known-hard reduces *to* your problem), then check you're not in a polynomial special case (tree/DAG/interval structure, k = 2, unit weights, matroid structure).
- **For heuristic "optimal" claims**: brute-force cross-validation on instances small enough to enumerate (n ≤ 12–15). Exactly optimal on all small random instances → credible; 10% off there → report it as approximate with that figure.
- **Constant-factor honesty**: any recommendation of a fancier structure over a flat array at n < 10⁴ should come with a measured (not asserted) win.
