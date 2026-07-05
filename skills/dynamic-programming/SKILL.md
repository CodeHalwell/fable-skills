---
name: dynamic-programming
description: Load when a problem involves optimizing/counting over sequences, subsets, intervals, or trees where brute force is exponential — recognizing DP in disguise, designing states, the four-step protocol, space optimization, top-down vs bottom-up, and the classic iteration-direction bugs.
---

# Dynamic Programming

Most DP mechanics are strong-model baseline (knapsack direction, loop-nesting semantics, LIS, interval DP, rolling-array hazards, lru_cache pitfalls). This sheet keeps the design protocol, the trigger table, and the debugging method.

## The protocol (run explicitly, in order)

1. **State** = the minimal summary of the past that determines the future — one precise sentence naming what's decided and what's pending ("dp[i][j] = min cost converting first i chars of A into first j of B"). A sentence needing "somehow" means redo. State design is 80% of the work; everything else is mechanical.
2. **Transition** by enumerating the *last* decision (usually easier than the first); every referenced state must be strictly smaller under a well-founded order — that order later dictates the loop order.
3. **Base cases** = identity elements: counting starts at 1 (one empty way), min at 0/∞ (guard the final answer against inf → report -1), max at 0/−∞ (0 silently permits the empty selection — check whether the problem allows it; all-negative inputs are the probe).
4. **Order/memoization**: bottom-up in dependency order; top-down when unsure (recursion cannot get the order wrong). Read the answer from the right cell — LIS is `max(dp)`, not `dp[-1]`.

DP = brute force + cache: if you can't write the exponential recursion, you can't write the DP — write it first; the only legal transformation is adding the cache (word-break style). Preconditions: optimal substructure (breaks on longest *simple* path — sub-solutions constrain each other) and overlapping subproblems (else plain D&C). Budget = #states × transition cost before coding.

## Trigger table (compressed)

Transform/align two sequences → dp[i][j] over prefixes. Capacity/budget selection, reach-sum-S, min-coins → knapsack dp[c]. Count ways → same with +. Burst balloons / matrix chain / merge stones / "last op on a range" → interval dp[l][r] by increasing length, deciding the *last* element removed (deciding the first fails because neighbors keep changing). Digits of N with property P → digit DP dp[pos][tight][P-state]. n≤20 visit-all/assignment → bitmask dp[mask][last]. Tree choices → post-order dp[v][took?]; answer-per-root → two-pass rerooting with prefix/suffix child aggregates. LIS family → sort one dimension, patience/bisect the other. Must-take-contiguous → Kadane. Game both-optimal → minimax margin per position. Segmentable-into-dictionary → dp over cut points.
Anti-triggers: uniform-cost explicit state graph → BFS; non-overlapping subproblems → D&C; a valid exchange argument → greedy.

## The recurring bugs (keep these sharp)

- **1-D knapsack direction:** descending capacity = 0/1 (each item frozen before it can feed itself); ascending = unbounded reuse. "Down for once, up for unlimited."
- **Loop nesting decides what you count:** items outer = combinations (coin order fixed); capacity outer = permutations (ordered sequences). Pick deliberately; verify against a hand-counted n=3.
- **State missing future-relevant information:** passes small tests, fails adversarial ones. Max-product needs max AND min ending at i (a negative flips them); paint-house needs last color; digit DP needs `tight`. Debug method: find the failing input, locate two histories that collapsed into one state but needed different futures — that difference is the missing component.
- **Interval DP order:** iterate by increasing r−l; the naive double loop reads uncomputed shorter intervals for some transitions and not others — which is why it's insidious. Check the first-iteration corner by hand, or go top-down.
- **Rolling arrays:** destroy path reconstruction — decide *before* rolling whether the output is a value or a path (full table, parent pointers, or Hirschberg for O(m)-space reconstruction). Edit-distance 1-row form needs the `prev_diag` juggle: one old-row cell must survive its own overwrite.
- **Python at scale:** `lru_cache` default maxsize=128 silently degrades DP to exponential — always `maxsize=None`; recursion dies ~10³ deep and is 3–5× slower than loops — derive top-down, convert bottom-up when n is large. `[[0]*m]*n` aliases one row n times (the "smeared table" — print it when confused).
- **Language-adjusted budgets:** n³ interval DP is fine at n=500 in C++, but ~10¹⁰ Python ops — pure-Python ceiling is n≈150–200; bitmask TSP 2²⁰·20·20 ≈ 4·10⁸ is fine compiled, n≤16 in Python. Budget in the language you'll ship.
- **Memo keys:** an accumulator in the key that's derivable from the indices = your state was wrong, and it can hide state-space blowup (O(n²) → O(n²·V)).

## Transition too slow? (state right, budget still fails)

Sliding-range min/max → monotonic deque (O(nk)→O(n)). Prefix-decomposable cost → running prefix aggregates (most O(n²) DPs collapse this way before any exotic trick). "Best previous with key < mine" → Fenwick/patience-bisect (O(n²)→O(n log n)). Linear-in-parameter transitions → convex hull trick / Li Chao; monotone split points → D&C optimization (O(kn²)→O(kn log n)); quadrangle inequality → Knuth–Yao (O(n³)→O(n²)). Submask sums total O(3ⁿ) via `sub=(sub-1)&mask` — budget with 3ⁿ, not 4ⁿ. Grid DPs wanting row+column range bests → maintain running row/col aggregates beside the table.

## Verification / self-check

- Re-state the state sentence; confirm the transition peeks at nothing outside it.
- #states × transition vs budget, with real numbers, in the shipping language.
- Cross-check against the exponential brute force on all inputs of size ≤ 8–10 (automate it).
- Probe the classic bug sites: reusable-item input (direction), all-negative (base cases), empty answer (identity), n = 0/1 boundaries, combinations-vs-permutations at n = 3.
- If space-optimized: confirm value-only output and that surviving cells are read before overwrite.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 12 baseline (cut/compressed), 1 partial (sharpened: language-adjusted budgets — the generic n≤500→n³ rule is compiled-language-only; Python ceilings are ~150–200 interval DP, n≤16 bitmask TSP), 0 delta.
- Opus 4.8 nailed direction semantics with the same mnemonic, nesting-decides-counting, max/min product state, patience LIS, last-balloon interval trick with the uncomputed-cell hazard, Hirschberg, prev_diag, lru_cache maxsize=128, base-case identities, ascending-mask topological validity, longest-simple-path substructure failure, the collapsed-histories debugging method — and listed *more* transition optimizations (CHT, Knuth–Yao, D&C opt) than the old text, now incorporated.
- Retained value: the four-step protocol as a forcing function, adversarial probes list, and language-adjusted budget corrections.
