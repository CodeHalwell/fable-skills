---
name: complexity-analysis
description: Loads when analyzing or predicting algorithm performance — deriving Big-O honestly (amortized/expected/worst-case), solving recurrences, judging whether a problem is NP-hard and what to do about it, explaining why the "slower" algorithm wins at real n (constants, cache), or auditing hidden complexity in library operations (list insert, string concat, dict resize).
---

# Asymptotic and Practical Complexity

Most of this domain is strong-model baseline (library-cost tables, master-theorem edge cases, pseudo-polynomiality, ReDoS, throughput anchors). This sheet keeps the anchors, the audit checklists, and the label discipline.

## Label discipline (the part people skip)

- Say **which** claim you're making: worst / expected (over the *algorithm's* randomness) / amortized (over an operation *sequence*). Throughput questions want amortized/expected; p99 SLOs and adversarial inputs want worst case. `dict` is O(1) expected, O(n) adversarial (Python randomizes string hashes; Java 8+ treeifies buckets; C++ `unordered_map` has no defense); `list.append` is O(1) amortized with O(n) resize spikes — pre-size on latency-critical paths.
- State complexity in the resource that dominates: an O(n) algorithm issuing n ORM queries is O(n) *round trips*; comparisons vs cache lines vs network calls vs tokens.
- Name the model before invoking a lower bound: Ω(n log n) binds comparison sorts (log₂ n! ≈ n log n); bounded integer keys escape via radix/counting; Ω(n) "must read the input" dies when an index/invariant already exists. A lower bound's deliverable is redirecting effort to a different model (index, hash, bits, randomization, approximation).

## Feasibility arithmetic (run before designing)

~10⁸–10⁹ simple ops/s/core; ~10⁷ touching random memory; ~10⁴–10⁵ syscalls/lock handoffs; ~10²–10³ network round trips. O(n²) at n=10⁵ = 10¹⁰ ops = ~10 s minimum — infeasible for a 1 s budget; the same at n=10³ is free. L1 ≈ 1 ns vs DRAM ≈ 100 ns is the ~100× hiding inside "O(1) access": same-O implementations differ 10–100× on access pattern, contiguous-scan crossovers sit around n ≈ 150–200 vs pointer-chasing n log n, and below n ≈ 100 the simplest code wins. Analyze to shortlist; measure to decide — and measure c₁, c₂ with a two-point microbenchmark, never guess them from the code's appearance.

## Recurrences (fallback: draw the tree)

Master theorem = three cached recursion trees; it is **silent** on: unequal splits (T(n/3)+T(2n/3)+n → tree → Θ(n log n) — any fixed-fraction split is n log n, which is why quicksort survives mediocre pivots and dies at (1, n−1)); subtract-and-conquer (T(n−1)+n → Θ(n²); 2T(n−1)+1 → Θ(2ⁿ)); non-polynomial gaps (2T(n/2)+n log n → Θ(n log² n)); root shrinkage (T(√n)+1 → m=log n → Θ(log log n)). Verify any claimed Θ by substitution. DP complexity = #states × work per state; O(nW) knapsack is **pseudo-polynomial** — exponential in W's bit length; at W=10⁹ the "polynomial" DP is a 10⁹-wide table (route: MIP, meet-in-the-middle, FPTAS).

## Hidden-cost cheat sheet (audit every hot-loop call)

`insert(0)`/`pop(0)` O(n) → deque · `x in list` O(n) → set · `str +=` O(total²) → join (the in-place optimization is refcount-fragile; never rely on it) · `bisect.insort` O(n) insert → SortedList · `np.append`/`pd.concat` per iteration O(n²) → accumulate, concat once · `iterrows` catastrophic constants → vectorize/`itertuples` · `sorted()` or `min`/`max` per iteration → heap / hoist · slices copy — `arr[mid:]` in recursion turns log into linear; pass indices · NumPy basic slicing is O(1) views but fancy indexing copies · `heapq.nsmallest(k)` O(n log k) beats full sort for small k · B-tree lookups cost O(log n) *IO-sized* hops — the log's base is the point on disk. Most real quadratic blowups are one hidden O(n) call inside an O(n) loop.

## NP-hard routing (and both failure directions)

Smells: subset selection under interacting constraints, routing/ordering with pairwise costs, partition-into-k with cross-interactions, SAT-like rule satisfaction. Menu: n≤~20–25 bitmask DP (Held–Karp O(n²2ⁿ)); n≤~40 meet-in-the-middle; structured instances → ILP/CP-SAT (routinely crush "NP-hard" at 10³–10⁵ variables, with proof or gap); monotone submodular + top-k → greedy at (1−1/e); metric TSP → NN + 2-opt (Christofides for the guarantee); else local search validated against exact solves on shrunk instances. Check the escape hatches *before* declaring hardness: tree/DAG/interval structure, a slack constraint, 2-SAT sufficing, TSP-on-a-line (sort), fixed tiny k. Equal failures: promising poly-time exactness on NP-hard structure, and calling assignment/max-flow/2-SAT/shortest-path "intractable".

## Sharp edges worth keeping explicit

- Regex on untrusted input is not bounded-cost: nested/ambiguous quantifiers (`(a+)+$` on `aaaa…X`) backtrack exponentially — a real DoS class; use RE2-style engines or timeouts, and never accept untrusted *patterns*.
- Python recursion dies at ~1000 frames; a 10⁶-deep linear recursion needs iteration or an explicit stack; `sys.setrecursionlimit` trades the exception for a possible C-stack segfault.
- Memoization's hidden bill: hashing O(len(key)) per lookup (tuple-of-slice keys can reintroduce the factor you removed), and `lru_cache(maxsize=None)` on a long-running service is a memory leak with a decorator's face. State key size and state count.
- Enumeration is Ω(#answers); if the answer set can be 10⁹ rows, change the question (count/sample/top-k/iterator). Counting can dodge it (DP, inclusion–exclusion).
- Amortized growth analysis: doubling → O(1)/push (copies sum to ≤ 2n); fixed-increment growth → O(n)/push; shrink with hysteresis (¼ occupancy) to avoid boundary thrash.
- Space audit alongside time: peak including temporaries (slices, sort copies), not just resident state — the O(n²) memo dies of memory before time.

## Verification / self-check

- Empirical scaling: time n, 2n, 4n; ratio ≈ 2 linear, ≈ 4 quadratic, 2+creep n log n; exponent = log₂(t₂ₙ/tₙ) from the large-n tail only. Measurement disagreeing with analysis means a hidden copy/rehash — believe the measurement, then find the term.
- Hot-loop library audit (table above); label the case (worst/expected/amortized) and match it to the caller's need.
- Hardness claims: name the known NP-hard problem that reduces *to* yours; heuristic "optimal" claims: brute-force cross-check at n ≤ 12–15 and report the measured gap.
- Adversarial probes for anything relying on randomness/hashing: sorted, reverse-sorted, all-equal, pathological keys.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed every probe: the full hidden-cost table with fixes, all five recurrence edge cases, pseudo-polynomiality with the W=10⁹ consequence, comparison-model escapes, throughput anchors and the 10-second O(n²)@10⁵ verdict, crossover ≈150, hash-flooding defenses per language, ReDoS with defenses, recursion/segfault behavior, memoization costs, doubling-ratio methodology, and exchange rates (9.6 bits/key Bloom, √L checkpointing).
- No substantive gaps found; the skill's value is the audit discipline (label the case, name the bound's model, feasibility arithmetic before design) and checklist completeness, so exposition was cut.
