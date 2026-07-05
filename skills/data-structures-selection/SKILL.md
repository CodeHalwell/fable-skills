---
name: data-structures-selection
description: Load when choosing a data structure for a task — matching access patterns to structures, hash map/heap/tree tradeoffs with real constants, top-k and streaming patterns, union-find triggers, bloom filters and sketches at scale, segment/interval trees, and when a plain sorted array wins.
---

# Data Structures Selection

Most of this catalog is strong-model baseline (two-heap median, B-tree arithmetic, Bloom sizing, DSU, SortedList internals). This sheet keeps the selection method, the constants, and the composite tricks.

## The method (this is the skill; structures are commodities)

- Select by **operation profile with frequencies**, not by name: list what the algorithm actually does (lookups? ordered iteration? min/max extraction? range queries? mid-stream inserts?), then buy exactly that. Paying for unused capabilities (balanced BST when order is never needed) costs constants and bugs for nothing.
- **Memory layout beats Big-O at real sizes**: a cache miss ≈ 100× an L1 hit; pointer-chasing structures run 5–50× behind their contiguous Big-O twins. Python object overhead is ~50–100 B/element — 3×10⁶ int64 ids = ~24 MB as a sorted numpy array vs ~200+ MB as a `set`, and one vectorized `np.searchsorted` over batched queries beats 5×10⁷ interpreter-dispatched `in` calls outright. Check n × bytes/element against memory before choosing; 10⁸ dict entries in 16 GB is not happening — that's the numpy/bitset/sketch signal.
- **Amortized ≠ smooth**: rehash/resize spikes are fine for throughput, fatal on latency paths and lock-held sections — pre-size or pick worst-case structures. Python offers no dict pre-size; bulk-build via `dict(zip(keys, vals))`.
- **Build-once/query-many → sorted array**, the most underrated entry: zero pointer chases, half the memory of any node-based tree, `bisect` in 4 lines, batch-vectorizable, ranges for free. Trees only earn their keep under interleaved inserts+queries; in Python that means `sortedcontainers.SortedList` (chunked array-of-arrays; `add` is ~O(√n) memmove in C, which still crushes pure-Python log-n pointer trees).
- **Composite profiles → composite structures**: O(1) lookup + order = dict + sorted structure; LRU = `OrderedDict`; streaming median = two heaps. Don't hunt one exotic structure when two boring ones glued together do it.

## Constants and formulas worth citing exactly

- B-trees: one page fetch buys a whole node of keys — branching ~500–1000 → 3–4 page reads for 10⁹ keys vs ~30 pointer hops for a binary tree; B+trees keep values in leaves (higher fan-out) and chain leaves for range scans. Write-heavy → LSM (sequential writes, compaction, per-SSTable Bloom filters); read-heavy point/range → B-tree. Ask the read/write ratio first.
- Bloom filter: ~9.6 bits/element for 1% FP, k ≈ 7 (k = (m/n)·ln2); no false negatives; can't delete (counting/cuckoo variants); FP rate degrades sharply past design capacity — size for peak. Count-Min for stream frequencies (overestimates only); HyperLogLog ~1.5 KB for ~2% distinct-count error. Never sketches for money, auth, or dedup-with-consequences.
- Heaps: top-k largest = size-k **min**-heap (the direction error evicts the largest of the kept set); `heapify` O(n) vs n pushes O(n log n); no arbitrary delete — lazy deletion (dead-set or stale-skip on pop; the standard Dijkstra idiom); tuple ties compare payloads and crash on uncomparable objects — insert a counter tiebreaker; `sorted()` beats popping a heap dry.
- DSU: triggers = incremental connectivity, Kruskal, merge-groups, streaming cycle detection (union returning False *is* the cycle detector); anti-trigger = deletions (can't un-merge; offline reversal or link-cut). Path halving + union by size = α(n); recursive `find` blows Python's stack — iterate.
- Bitsets: subset-sum reachability `reach |= reach << coin` = ~64× word-parallel speedup; submask iteration `sub = (sub-1) & mask` totals O(3ⁿ) over all masks — budget with that number; `int.bit_count()` (3.10+) for popcount.

## Hash pitfalls (the expensive ones, compressed)

Mutation during iteration (collect keys first) · mutable keys / `__hash__` over mutable fields → unfindable entry + leak (tuple/frozenset the key; `@dataclass(frozen=True)` gives consistent eq+hash — defining `__eq__` alone sets `__hash__ = None`) · `hash()` is per-process randomized for str/bytes — never persist or send it (use hashlib) · iteration order: Python 3.7+ insertion-ordered, Go deliberately randomized, C++/Rust arbitrary — sort explicitly when order matters · float keys miss (`0.1+0.2 != 0.3`) — quantize or fixed-point · adversarial collisions degrade to O(n): C++ `unordered_map` has no defense — seeded hash for untrusted keys.

## Selection judgment calls that go beyond the defaults

- Fenwick when the op is invertible (sum/XOR): 15 lines, tiny constant. Segment tree for min/max/gcd or lazy range updates. **All queries known upfront → sort + sweep instead** — most "I need an interval tree" moments are sweep-line problems, simpler and faster.
- Trie vs `bisect` over sorted strings: the latter answers most prefix queries in 3 lines without the per-node memory hog.
- Dense integer keys in a known range → plain array/bitset — the fastest "hash map" ever shipped. 2-D grid visited state → boolean list-of-lists, 3–5× faster than a set of tuples (per-check tuple hash + allocation).
- `Counter.most_common(k)`, `Counter & Counter` — reach for them before hand-rolling top-k/multiset ops.
- Key design = state design: `frozenset` keys collapse order-dependent states; if the future depends on sequence, the collapsed key silently merges distinct states and the search returns wrong answers only on inputs where order mattered.
- Priorities mutated in place silently break heap invariants (heapq never re-checks) — lazy delete + fresh push.
- Four parallel dicts keyed identically → one dict of dataclasses: four lookups where one suffices plus update-consistency bugs.

## Verification / self-check

- Re-confirm each operation's cost *in the chosen library's implementation* (SortedList.add ~O(√n), not O(log n) — usually fine, claim it correctly).
- n × bytes/element vs available memory; amortized spikes vs the latency budget; adversarial keys vs the hash.
- Trace one insert + one query by hand at size 3 — most selection errors surface immediately; benchmark at target n, not n=100 (rehash/GC/cache effects don't extrapolate).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 13 claims: 13 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed everything: min-heap direction for top-k, push-then-migrate median insertion, the 24 MB-vs-200 MB numpy-vs-set analysis with the batching argument, B-tree page math and LSM tradeoff, 9.6 bits/7 hashes Bloom sizing with saturation behavior, DSU incl. path halving and the union-return cycle bit, heapq gotchas, hash randomization/iteration-order per language, Fenwick-vs-segment invertibility rule and the sweep-line escape, SortedList internals with the O(√n) add, bitset DP at ~64×, and O(3ⁿ) submasks.
- No substantive gaps; retained value is the selection method (profile → layout → constants), the exact formulas, and checklist completeness.
