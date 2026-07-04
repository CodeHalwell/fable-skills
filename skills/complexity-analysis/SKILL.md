---
name: complexity-analysis
description: Loads when analyzing or predicting algorithm performance — deriving Big-O honestly (amortized/expected/worst-case), solving recurrences, judging whether a problem is NP-hard and what to do about it, explaining why the "slower" algorithm wins at real n (constants, cache), or auditing hidden complexity in library operations (list insert, string concat, dict growth).
---

# Asymptotic and Practical Complexity

## Core mental model

1. **Big-O is a family of claims — say which one you're making.** Worst-case, expected (over randomness *in the algorithm*, e.g. hashing/quicksort pivots), and amortized (averaged over an operation *sequence*) answer different questions. `dict` lookup is O(1) expected, O(n) worst-case (adversarial collisions); `list.append` is O(1) amortized, O(n) on a resize. For throughput, amortized/expected is what matters; for per-operation latency SLOs and adversarial inputs, worst case is what matters. Choosing the wrong one is how "O(1)" systems miss p99.
2. **Asymptotics predict *scaling*, constants predict *time*.** O(n²) with a tight cache-friendly inner loop beats O(n log n) with pointer chasing until n is thousands. Real sort implementations use insertion sort below ~16–64 elements for exactly this reason (Timsort, introsort cutoffs). The expert question is never "which is asymptotically better" but "where's the crossover, and which side of it is my n?"
3. **Memory access dominates modern constants.** L1 ≈ 1 ns, main memory ≈ 100 ns, so an O(n) pass with random access can cost more than an O(n log n) pass with sequential access. Contiguous + predictable (arrays, sorting then scanning) beats asymptotically-equal linked/hashed structures at practical sizes. This is why "sort then group" often beats "hash then group" and why B-trees beat binary trees on disk.
4. **Lower bounds tell you when to stop optimizing the algorithm and change the problem.** Comparison sorting is Ω(n log n) (log₂(n!) comparisons needed to distinguish n! orderings) — beating it requires *non-comparison* information (radix/counting sort on bounded integer keys: O(n·k)). Searching an unsorted array is Ω(n) by adversary argument: any element you didn't inspect could have been the target. When you hit a lower bound, the wins left are constants, or restructuring so the bound doesn't apply (index it, sort it once, maintain an invariant).
5. **NP-hardness is a routing decision, not a death sentence.** Recognizing an NP-hard core early redirects you from "find the efficient algorithm" (doesn't exist as far as anyone knows) to the productive menu: exact search that's fine at your n (n ≤ 20 → bitmask DP 2ⁿ·n; n ≤ 40 → meet-in-the-middle), ILP/CP-SAT solvers (routinely exact for thousands of variables of *structured* problems), approximation with guarantees, or local-search heuristics with an evaluation harness.

## Decision frameworks

### Recurrences
- **Master theorem** for T(n) = a·T(n/b) + f(n): compare f(n) to n^(log_b a). Leaf-dominated (f smaller) → T = Θ(n^(log_b a)); balanced (equal, up to log factors) → multiply by log n; root-dominated (f bigger + regularity) → T = Θ(f). Mechanically: mergesort a=2,b=2,f=n → n^1 = n → Θ(n log n); Karatsuba a=3,b=2,f=n → n^1.585 dominates → Θ(n^1.585); binary search a=1,b=2,f=1 → Θ(log n).
- **When the master theorem does NOT apply**: unequal subproblem sizes (T(n)=T(n/3)+T(2n/3)+n — use a recursion-tree: depth log n, n work per level → Θ(n log n)); subtract-and-conquer (T(n)=T(n−1)+n → sum the series: Θ(n²)); non-polynomial gaps (f = n log n vs n: case 3 fails its hypothesis — answer is Θ(n log² n), get it from the tree); randomized splits (quicksort — expectation argument, not master theorem).
- Default fallback: **draw the recursion tree, sum per-level work, multiply by depth pattern**; then verify by substitution. The tree never lies; the theorem is just three cached trees.
- DP complexity = (#distinct states) × (work per state). Count states honestly: bitmask over n items = 2ⁿ states; intervals = O(n²); (index, remaining budget) = O(n·B) — which is *pseudo*-polynomial: B is exponential in its bit-length. Knapsack's O(n·B) DP does not contradict its NP-hardness.

### NP-hard recognition and the response menu
Smells: "choose a subset maximizing X under interacting constraints" (knapsack-with-conflicts, set cover), "order/route visiting all items" (TSP, scheduling with precedence+deadlines), "partition into k groups minimizing interactions" (graph coloring, clustering with hard constraints), "satisfy all these boolean-ish rules" (SAT-like). Response by instance size and need:
| Situation | Reach for |
|---|---|
| n ≤ ~20–25 | Exact exponential: bitmask DP (Held–Karp TSP O(n²2ⁿ)), branch-and-bound |
| Structured, ≤ ~10³–10⁵ vars, need (near-)optimal + guarantee | ILP (HiGHS/Gurobi) or CP-SAT (OR-Tools); modern solvers crush many "NP-hard" instances |
| Objective is monotone submodular + cardinality constraint | Greedy: (1−1/e)-approximation, provably near-best-possible; usually the right answer to "pick k items" |
| Metric TSP-like | Christofides 1.5x or, practically, nearest-neighbor + 2-opt/Or-opt |
| Huge, time-boxed, no guarantee needed | Local search / simulated annealing / LNS, validated against exact on small instances |
- Special-case escape hatches before declaring hardness: is the graph a tree/DAG/interval graph (many NP-hard problems become poly there)? Is a constraint actually always slack? Is 2-SAT enough (poly) rather than 3-SAT? Is the "TSP" on a line (trivial)?

### Crossover estimation (constants and cache)
- Estimate the crossover: if algorithm A is c₁·n² (say 1 ns/op, contiguous) and B is c₂·n log n (say 20 ns/op, pointer-chasing), A wins while n < (c₂/c₁)·log n ≈ 20·log n → n ≲ ~150–200. Measure c₁, c₂ with a microbenchmark at two sizes; don't guess.
- Branch mispredictions and cache misses commonly contribute 10–100× to constants: that's the entire gap between "same Big-O" implementations. When two options tie asymptotically, pick the one that scans memory forward.

### Space–time tradeoffs (state the exchange rate)
Hash-index a relation: O(n) space buys O(1) probes replacing O(n) scans — worth it after ~2 lookups. Memoize: space = #states, time drops from tree to DAG size (Fibonacci: 2ⁿ → n). Precomputed prefix sums: O(n) space, range sums O(n)→O(1). Bloom filter: bits-per-key ≈ 10 buys ~1% false positives, no false negatives — right when a miss is cheap to double-check. Recompute-vs-store (activation checkpointing): pay ~1.3–2× time to cut memory ~√-fashion. Always name what's bought, what's paid, and the break-even usage count.

## Failure modes & pitfalls

- **Quoting O(1) dict/set behavior where the worst case is the requirement** — or missing that untrusted keys can force collisions (hash-flooding; Python randomizes string hashes for this reason). Conversely, avoiding dicts "because worst case O(n)" in offline analytics is cargo-cult.
- **Hidden quadratics in innocent-looking library code** (the highest-frequency real-world class):
  - `s += chunk` for strings in a loop: strings are immutable → each += copies → O(total²). Use `''.join(parts)`. (CPython sometimes optimizes in-place concat; never rely on it — it vanishes across refcount conditions and implementations.)
  - `list.insert(0, x)` / `list.pop(0)` in a loop: each is O(n) (shifts everything) → O(n²). Use `collections.deque` (O(1) both ends).
  - `x in list` inside a loop: O(n) each → O(n²); make a set first (one O(n) pass).
  - `list.remove(v)` / `del d[k]` while iterating: besides the runtime error risk, remove is O(n) search + O(n) shift.
  - `np.append`/`pd.concat` in a loop: full reallocation each time → O(n²) and memory churn. Accumulate in a Python list, then one `np.concatenate`/`pd.concat` at the end.
  - Repeated `sorted()` inside a loop to get the max/min k: use `heapq.nlargest` or maintain a heap.
- **Amortized ≠ smooth**: `list.append` occasionally does an O(n) resize; a latency-critical loop sees rare big stalls that "O(1) amortized" hides. Pre-size (`[None]*n`, `dict.fromkeys`), or accept the spikes knowingly. Same for dict growth: inserting n items triggers ~log-many rehash events, each O(current size).
- **Slicing and copying inside recursion**: `binary_search(arr[mid:])`-style code turns O(log n) into O(n) *per level* → O(n log n) or worse. Pass indices, not slices. Same bug: `sum(lst[:i])` in a loop.
- **Believing the master theorem covers everything**: applying it to T(n) = 2T(n−1) + 1 (it's subtract-not-divide → Θ(2ⁿ)) or T(n)=T(√n)+1 (change variables m = log n → Θ(log log n)). If b doesn't divide n's *size* multiplicatively, the theorem is silent.
- **"It's polynomial" claims via pseudo-polynomial DPs or n^O(log n) constructions** — knapsack O(nW) is exponential in input bits; be precise or the scaling surprise arrives with real data.
- **Misreading Ω lower bounds as unconditional**: Ω(n log n) sorting is for *comparison* sorts on arbitrary orderable keys. Bounded ints → radix O(n·k) beats it legitimately. Similarly "you must read all input" Ω(n) dies when an index/invariant already exists. Check the bound's *model* before invoking it.
- **Assuming the expected case without its precondition**: quicksort's O(n log n) is expected over *random pivots*; deterministic first-element pivot on sorted input is O(n²) (and library introsorts guard with heapsort fallback for this reason). Hash tables' O(1) assumes hashing spreads keys — timestamps rounded to the hour, or user IDs sharing a stride, can cluster in some hash designs.
- **Optimizing the O without profiling the constant**: replacing an O(n²) that runs on n=40 (microseconds) while the actual cost is an O(n) network call per element. Complexity analysis ranks *terms*; a profiler ranks *reality*. Do the asymptotics to pick candidates, then measure.
- **Ignoring output size**: "enumerate all pairs/subsets matching X" is Ω(#answers) — no algorithm beats its own output size; if the answer can be 10⁹ rows, the fix is changing the question (count, sample, or top-k), not the algorithm.

## Worked micro-examples

**1. String concat, quantified.** Building a 10⁶-char string from 10⁴ pieces of 100 chars: naive `+=` copies ~Σᵢ(100·i) ≈ 100·(10⁴)²/2 = 5×10⁹ char-copies; `''.join` copies each char once ≈ 10⁶. That's a ~5000× gap from one idiom — visible as "the report generator takes minutes".

**2. Recursion tree where the master theorem is silent.** T(n) = T(n/3) + T(2n/3) + cn. Every level's work sums to cn (pieces partition n); the deepest path shrinks by 2/3 per step → depth log_{3/2} n ≈ 1.71·log₂ n. Total Θ(n log n). Bonus insight: unbalanced splits change only the *constant*, not the n log n class — this is why quicksort survives mediocre pivots (any fixed-fraction split works) but dies on 1/(n−1) splits (depth n → Θ(n²)).

**3. NP-hard recognition redirect.** "Assign 200 tasks to 20 workers; each task has a subset of qualified workers; some task pairs can't share a worker; minimize max load." Conflicts + makespan = graph-coloring/scheduling hybrid → NP-hard; don't promise an efficient exact algorithm. Route: CP-SAT model (boolean x[t,w], AddAtMostOne per conflict pair per worker, minimize max load) solves this size in seconds and proves optimality or a gap; fallback greedy (order tasks by fewest qualified workers, assign to least-loaded feasible worker) + local search (swap/2-move) if the solver can't be a dependency. This routing — hardness spotted, solver-or-heuristic chosen, guarantee stated — is the expert deliverable.

## Verification / self-check

- **Empirical scaling check**: time the code at n and 2n (and 4n). Ratio ≈ 2 → linear; ≈ 4 → quadratic; ≈ 2 + slow creep → n log n. If the measured exponent disagrees with your analysis, the analysis missed a hidden copy/rehash/library call — believe the measurement, then find the term.
- **Recurrence check by substitution**: plug the claimed Θ back into the recurrence and confirm both sides balance; also sanity-check T at n=1,2,4 by hand.
- **Audit every library call in the hot loop** for its true cost (`in` on list vs set, `insert(0)`, slicing, `+=` on str/bytes, DataFrame ops inside loops); the loop body's asymptotics is the *product* of loop count and the worst call inside.
- **State the case**: before reporting a bound, label it worst/expected/amortized and check the label matches the caller's need (throughput vs p99 vs adversarial input).
- **For hardness claims**: name the known NP-hard problem your problem contains (reduction direction: known-hard reduces *to* yours), and double-check you're not in a poly special case (tree/DAG/interval structure, k=2, unit weights).
- **For "optimal" claims from heuristics**: cross-validate on instances small enough for brute force (n ≤ 12–15) — a heuristic that's exactly optimal on all small random instances is credible; one that's 10% off there should be reported as approximate.
