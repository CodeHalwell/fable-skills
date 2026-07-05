---
name: algorithm-design
description: Load when designing an algorithm from scratch — competitive-programming problems, interview questions, or production code where the naive approach is too slow. Covers problem classification, invariant-first design, greedy proof obligations, reductions to known problems, two-pointer/sliding-window/monotonic-stack triggers, randomization, and systematic boundary handling.
---

# Algorithm Design

Most techniques here are strong-model baseline (binary search on the answer, MITM, monotonic stacks, circular Kadane, exchange arguments). This sheet keeps the trigger tables, the proof-obligation discipline, and the boundary conventions.

## Order of operations (the discipline, not the techniques)

1. **Budget from constraints before designing:** n≤20 → 2ⁿ/bitmask; n≤500 → n³; n≤5000 → n²; n≤10⁵–10⁶ → n log n or n; n≥10⁹ → log n / O(1) math / binary search on the answer / matrix power. If the current idea exceeds the budget, the *approach* is wrong — don't micro-optimize it.
2. **Reduce before inventing:** sorting in disguise? shortest path on an implicit graph? matching/flow? interval scheduling? known NP-hard core (then say so and route to bitmask DP / pseudo-poly DP / approximation — never ship a wrong greedy silently)?
3. **State the invariant in one sentence** ("window [l,r] is the maximal valid window ending at r"); it is the correctness proof and the loop-body checklist. Can't state it → you have vibes, not an algorithm.
4. **Hunt the monotone quantity when stuck** — binary search, two pointers, window, monotonic stack, and greedy all rest on one.

## Trigger table (one-liners)

Min/max X with monotone feasibility → binary search on the answer (only need a checker; verify check(x) ⇒ check(x±1) in the right direction first — "partition into exactly k parts" often *isn't* monotone). Longest/shortest contiguous with shrink-monotone P → sliding window (negatives break sum-windows → prefix sums + hashmap `count += seen[P−k]`, seed `seen[0]=1`). Sorted + pair relation → two pointers (each advance must provably abandon no future answer). Next greater / largest rectangle / spans → monotonic stack; window max/min → monotonic deque. k-th smallest → quickselect or size-k heap. Cross-midpoint interactions with O(n) combine → D&C (inversions = merge sort's swaps; if combine needs full sub-solutions, D&C won't beat brute force). Range sums no updates → prefix sums (half-open); many range updates one read → difference array; 2-D → inclusion–exclusion (the sign pattern is where bugs live). n≈40 exact target → meet in the middle (2·2²⁰ + hash join). "Exactly k" → atMost(k) − atMost(k−1). Cyclic → concatenate conceptually or wraparound case (circular Kadane = max(normal, total − min-subarray), and all-negative must return max element, not the empty 0). Adversarial input / stream sampling / cheap verification → randomization (reservoir k/i; randomize rolling-hash base at runtime; prefer Las Vegas when you can verify).

## Greedy: proof obligations, not vibes

- Exchange argument or it's conjecture: swap OPT's first divergent choice for greedy's, show no worse, induct. If you can't execute it mentally in ~30 s, brute-force-check against an exponential oracle on all inputs of size ≤ 8 *before* trusting (5-line itertools harness — write it).
- Cached sort-key proofs: interval scheduling → earliest finish; weighted completion → p_i/w_i; concatenate-largest-number → comparator a+b > b+a. Pattern: prove the *adjacent* swap never helps, validating the sort order.
- Cached traps: coin change greedy fails off canonical systems ({1,3,4}, target 6: greedy 4+1+1 vs 3+3); 0/1 knapsack by density is wrong (fractional is the greedy-safe variant); local-cheapest fails whenever choices couple through a shared budget.
- Matroid smell = decisive shortcut: feasible sets closed under subsets + exchange property → greedy-by-weight is provably optimal (spanning trees; "k items with distinct labels"; unit-time deadline scheduling — where DSU on "latest free slot ≤ deadline" gives O(n α(n))).

## Boundary discipline (the off-by-one system)

- One binary-search template, never improvised: `lo, hi = 0, n`; `while lo < hi: mid = (lo+hi)//2; hi = mid if ok(mid) else (lo := mid+1)`; answer `lo`; check `lo == n` (no true exists) before indexing. The three classic bugs: `<=` with exclusive updates (infinite loop); floor-mid with `lo = mid` (infinite loop at hi = lo+1 — surviving-lo needs ceil-mid); forgetting the no-true sentinel.
- Reals: `while hi - lo > 1e-9` can spin forever near large magnitudes (ULP > ε) — iterate `for _ in range(100)` instead, unconditionally safe.
- Conventions fixed once per solution, written as a comment: half-open `[l, r)` everywhere (length r−l, seamless tiling, empty = l==r); `prefix[i]` = sum of first i (so sum(l,r) = P[r]−P[l], no −1 anywhere). Test mentally at n = 0, 1, 2 before running; check length-1 and full-range queries and that adjacent ranges tile.
- Tie semantics in monotonic stacks are a spec decision, not a default: pop on `<` vs `≤` decides them, and largest-rectangle needs one strict + one non-strict side to avoid double-counting equal-height bars.
- When speccing Python for C++/Java: overflow-safe `mid = lo + (hi-lo)/2`, cast before multiplying, `(a-b) % MOD` needs +MOD, modular division = `pow(b, MOD-2, MOD)`. Sort-by-angle/ratio comparisons in integers via cross-multiplication, floats for output only.

## When stuck: the unstick sequence

(1) Recompute the budget — n log n with no sort in sight means binary search on the answer or a heap/window. (2) Sort by each plausible key and stare. (3) Ask "what is the last decision?" (DP) and "what is the state graph?" (BFS/Dijkstra). (4) Delete one constraint, solve, reintroduce — the delta names the technique. (5) Solve n = 1,2,3 by hand and diff. (6) Invert: count/remove the complement ("min removals" = n − "max kept", and the kept version is often a classic).

## Verification / self-check

- Walk the invariant: initialization, maintenance, termination ⇒ answer.
- Complexity vs budget with real numbers (10⁵·17 ≈ 1.7M fine; (10⁵)² = 10¹⁰ dead).
- Adversarial micro-tests: empty, single element, all-equal (ties), sorted/reverse, negatives if the domain allows, extreme constraint values.
- Re-read the statement after solving — the highest-frequency expert failure is solving the adjacent problem (contiguous vs not, at-most vs exactly, strict vs non-strict).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed everything: budget mapping, split-array binary search with bounds and checker proof, all three binary-search bug classes, the {1,3,4} greedy trap with exchange-argument execution and stress-test fallback, negatives-break-windows with the prefix-hashmap fix, monotonic-stack tie semantics incl. the histogram strict/non-strict asymmetry, MITM, exactly-k subtraction, fixed-iteration real bisection, deadline-scheduling greedy with DSU, circular Kadane's all-negative edge case, and the half-open/prefix conventions.
- No substantive gaps; the retained value is the ordering discipline (budget → reduce → invariant), the proof-obligation gate on greedy, and trigger-table completeness.
