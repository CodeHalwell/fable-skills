---
name: data-structures-selection
description: Load when choosing a data structure for a task — matching access patterns to structures, hash map/heap/tree tradeoffs with real constants, top-k and streaming patterns, union-find triggers, bloom filters and sketches at scale, segment/interval trees, and when a plain sorted array wins.
---

# Data Structures Selection

## Core mental model

- **Select by operation profile, not by name.** List the operations the algorithm actually performs, with frequencies: lookups? ordered iteration? min/max extraction? range queries? insertions mid-stream? Then pick the cheapest structure covering exactly that profile. Paying for capabilities you don't use (a balanced BST when you never need order) costs constant factors and code complexity for nothing.
- **Big-O lies about constants; memory layout tells the truth.** A cache miss costs ~100× an L1 hit. An array scan of 100 elements typically beats a hash lookup chain that misses cache twice. Pointer-chasing structures (linked lists, naive BSTs) are 5–50× slower than their Big-O twin in contiguous memory. This is *the* reason real systems look different from textbooks.
- **Amortized ≠ smooth.** Dynamic arrays and rehashing hash maps have O(1) amortized ops with occasional O(n) spikes. Fine for throughput, potentially fatal for per-operation latency budgets (real-time, lock-held sections). If tail latency matters, pre-size or pick a structure with worst-case bounds.
- **The sorted array is the most underrated structure in the catalog.** For build-once/query-many workloads it beats everything: binary search with zero pointer chases, perfect cache behavior, half the memory of any node-based tree, and `bisect` is 4 lines. Reach for trees only when interleaved inserts and queries force you to.
- **Composite structures solve composite profiles.** Need O(1) lookup AND ordered iteration? dict + sorted structure. LRU cache = hash map + doubly linked list (`OrderedDict`). Streaming median = two heaps. Don't hunt for one exotic structure when two boring ones glued together do it.

## The selection table

| Dominant operation profile | Structure | Real-world notes on constants |
|---|---|---|
| Lookup/insert/delete by key, no order needed | Hash map | O(1) expected; watch rehash spikes and iteration-order assumptions |
| Membership only, huge scale, false positives OK | Bloom filter | ~10 bits/element for 1% FP; see sketches section |
| Ordered iteration + point lookup, build-once | Sorted array + `bisect` | Beats trees by 2–10× on query; O(n) insert is the tax |
| Ordered ops with interleaved inserts | Balanced BST / skip list; in Python: `sortedcontainers.SortedList` | SortedList (chunked array-of-arrays) is cache-friendly and usually fastest in Python |
| Repeated min/max extraction, insert-heavy | Binary heap (`heapq`) | Array-backed, cache-decent; no efficient arbitrary delete — use lazy deletion |
| Top-k of a big/streaming set | Size-k min-heap (for top-k largest) | O(n log k), memory O(k); `heapq.nlargest` does exactly this |
| FIFO/sliding window ends | `collections.deque` | O(1) both ends; never use `list.pop(0)` — it's O(n) |
| Connectivity queries under union-only merging | Union-find (DSU) | Near-O(1) amortized with path compression + union by rank |
| Range aggregate + point/range updates | Segment tree / Fenwick (BIT) | Fenwick: ~15 lines, prefix-sum-invertible ops; segment tree: any associative op, lazy propagation for range updates |
| Stabbing queries ("which intervals contain point x") | Interval tree; or sort + sweep if offline | If queries can be sorted/batched, a sweep line kills the tree |
| String prefix queries, autocomplete | Trie | Memory hog (per-node dict/array); sorted array of strings + `bisect` on prefixes often suffices |
| Dense integer keys in a known range | Plain array/bitset | The fastest "hash map" ever shipped |
| Disk/large-memory ordered storage | B-tree/B+tree | See "why databases choose B-trees" |

## Hash map pitfalls (the expensive ones)

- **Mutation during iteration.** Python raises `RuntimeError` for dict size changes mid-iteration; C++ invalidates iterators on `unordered_map` rehash; Java throws `ConcurrentModificationException` (best case) or silently misbehaves. Correction: collect keys first (`for k in list(d)`) or build a new map.
- **Mutable keys.** A key mutated after insertion changes its hash and becomes unfindable while still occupying a slot — no error, just a leak plus a lookup miss. Python blocks lists/dicts as keys, but a custom class with `__hash__` over mutable fields recreates the bug. Rule: hash only immutable identity; if you need to key by a list, use `tuple(x)`; by a set, `frozenset(x)`.
- **`__eq__` without `__hash__` (Python).** Defining `__eq__` on a class sets `__hash__ = None` — instances become unhashable, surprising anyone adding them to a set later. `@dataclass(frozen=True)` gives you both, consistently.
- **Adversarial worst case.** All keys colliding degrades to O(n) per op — a real DoS vector for services hashing user-controlled strings. Python randomizes string hashes per process (note: `hash("x")` differs across runs — never persist or send `hash()` values); Java 8+ tree-ifies hot buckets; C++ `unordered_map` has no defense — use a seeded/siphash-based hash for untrusted keys.
- **Iteration-order assumptions.** Python dicts preserve insertion order (guaranteed 3.7+); Go randomizes deliberately; C++/Rust HashMap order is arbitrary and seed-dependent. Code that "works" by iterating a hash map in a meaningful order is a portability landmine — sort explicitly when order matters.
- **Float keys.** `0.1 + 0.2 != 0.3` means computed float keys miss. Quantize (`round(x, 9)`) or use integer fixed-point keys.
- **Rehash cost at scale.** Inserting 10M items triggers ~24 rehashes copying everything. Pre-size when the count is known: C++ `reserve(n)`, Java `new HashMap<>(n * 4 / 3)`; Python offers no pre-size — for bulk builds, `dict(zip(keys, vals))` in one shot is meaningfully faster than a loop.

## Heap patterns

- **Top-k largest of n:** min-heap of size k — push, and pop when size exceeds k. O(n log k), not O(n log n). Inverting the comparison direction (max-heap for top-k largest) is the classic sign error: you'd evict the *largest* instead of the smallest of the kept set.
- **Streaming median:** max-heap `lo` (lower half) + min-heap `hi` (upper half), rebalance to |len(lo) − len(hi)| ≤ 1. Python has no max-heap: negate values on `lo`. Push to `lo` first, then move `lo`'s max to `hi`, then rebalance sizes — this ordering avoids the "new element on wrong side" bug.
- **Lazy deletion (the workaround for heapq's missing `remove`):** to delete arbitrary items, don't. Mark them dead (a set of dead ids, or a version counter per key) and discard stale entries when they surface at the top: `while heap and heap[0] is stale: heappop(heap)`. This is also *the* standard Dijkstra idiom: push duplicates, skip on pop if `dist[u] < d`. Memory grows by the duplicate count — fine when duplicates are O(edges).
- **Tuple ties:** `heapq` compares tuples element-wise; if priorities tie, it compares payloads — crashing on uncomparable objects. Insert a tiebreaker counter: `heappush(h, (priority, next(counter), item))`.
- **Heapify is O(n), not O(n log n).** Building a heap from a list: `heapq.heapify(lst)` beats n pushes. If you need all elements sorted anyway, `sorted()` beats heap-popping n times (better constants, same complexity).

## Trees, skip lists, B-trees — and why databases pick B-trees

- In-memory ordered maps: red-black/AVL trees, skip lists, and B-trees are all O(log n). Skip lists win for lock-free concurrency (Java's `ConcurrentSkipListMap`, Redis sorted sets) because insertion touches few nodes probabilistically. Binary trees lose in practice because each comparison is a random pointer dereference — a cache miss.
- **B-trees win on storage and even in-memory because of transfer granularity.** Disk reads are 4KB pages, cache reads are 64B lines. A B-tree node packs hundreds of keys into one page, so one fetch buys ~9 comparisons' worth of progress (branching factor ~500 → log₅₀₀(10⁹) ≈ 3–4 page reads for a billion keys, vs ~30 for a binary tree). B+trees additionally chain leaves for fast range scans and keep values out of interior nodes so more keys fit per page. Same logic at cache-line scale explains why in-memory B-trees (Rust `BTreeMap`, node size tuned to lines) beat red-black trees.
- **LSM-trees vs B-trees (know when the DB flips):** write-heavy workloads (logs, time series) favor LSM (RocksDB, Cassandra) — sequential writes, deferred merge; read-heavy point/range lookups favor B-trees (Postgres, MySQL InnoDB). If asked to pick a storage index, ask the read/write ratio first.
- In Python, skip the whole debate: `sortedcontainers.SortedList/SortedDict` (pure Python, chunked arrays) empirically outperforms tree implementations for nearly all sizes. In C++, `std::map` is a red-black tree; prefer `std::unordered_map` unless order/range ops are needed, and consider `boost::flat_map` (sorted vector) for read-mostly.

## Union-find: triggers and implementation

Triggers: "are X and Y connected", incremental edge additions with connectivity queries, Kruskal's MST, "merge accounts/groups", cycle detection in an undirected graph while streaming edges, grid percolation. Anti-trigger: edge *deletions* — DSU can't un-merge; you need offline reversal tricks or a different structure.

```python
parent = list(range(n))
size = [1] * n
def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]   # path halving: as good as full compression, no recursion
        x = parent[x]
    return x
def union(a, b):
    ra, rb = find(a), find(b)
    if ra == rb: return False           # already connected — this return value IS your cycle detector
    if size[ra] < size[rb]: ra, rb = rb, ra
    parent[rb] = ra
    size[ra] += size[rb]
    return True
```
Both optimizations together give α(n) amortized (effectively constant). Path compression alone is enough in practice; skipping *both* degrades to O(n) chains. Recursive `find` overflows Python's stack on ~1000-deep chains — use the iterative form above.

## Bloom filters and sketches (approximate structures for scale)

- **Bloom filter:** set membership, no false negatives, tunable false positives. Sizing: m/n ≈ 10 bits per element → ~1% FP with k = 7 hash functions (optimal k = (m/n)·ln 2). Use for: "definitely not present" fast paths — cache admission, LSM SSTable filters, "have we crawled this URL". Cannot delete (use counting Bloom or cuckoo filter if you must). Pitfall: FP rate degrades badly past design capacity — size for peak n, not average.
- **Count-Min Sketch:** approximate frequencies of stream items in sublinear memory; overestimates only. Use for heavy hitters/rate limiting at scale.
- **HyperLogLog:** distinct-count in ~1.5KB with ~2% error (Redis `PFCOUNT`). Use whenever the question is "how many unique" and exactness isn't contractual.
- Decision rule: reach for a sketch when n × (bytes per exact entry) exceeds comfortable memory AND the consumer tolerates quantified error. Never for money, auth, or dedup-with-consequences.

## Segment trees, Fenwick, interval structures

- **Fenwick (BIT)** when the operation is invertible (sum, XOR): point update + prefix query in O(log n), ~15 lines, tiny constant. Range-sum query = two prefix queries.
- **Segment tree** when you need min/max/gcd (non-invertible) or range updates (lazy propagation). 4n array sizing, iterative versions are faster but recursive is far easier to get right.
- **Recognition:** "range query + updates interleaved" → segment tree family. "All queries known upfront" → consider offline sorting/sweep line instead; it's often simpler AND faster.
- **Interval trees vs the cheap alternative:** for "which intervals contain point x" queries arriving online, an interval tree (or `IntervalTree` from the `intervaltree` package) is right. If intervals and queries are both known, sort endpoints and sweep — O((n+q) log(n+q)) with trivial code. Most "I need an interval tree" moments are actually sweep-line problems.

## Bit manipulation structures

- **Bitset as a set of small ints:** Python arbitrary-precision ints make elegant bitsets: `mask |= 1 << x`, test `mask >> x & 1`, iterate set bits via `while m: low = m & -m; ...; m ^= low`. Popcount: `bin(m).count("1")` or 3.10+ `m.bit_count()` (much faster).
- **Bitset DP acceleration:** subset-sum reachability in O(n·maxsum/64): `reach |= reach << coin`. This one trick turns 10⁸ operations into 10⁶ — remember it exists.
- **Bitmask as dict key for subset DP:** `dp[mask]` over 2^n subsets; iterate submasks with `sub = mask; while sub: ...; sub = (sub - 1) & mask` — total O(3^n) over all masks, a fact worth knowing when budgeting.

## Failure modes & pitfalls (cross-cutting)

- Using `list.insert(0, x)` / `pop(0)` as a queue — O(n) each; that "mysteriously slow BFS" is usually this. Use `deque`.
- Choosing a heap when you need "min AND delete arbitrary" — that's a sorted structure or lazy deletion, not more heap.
- Reaching for a trie/suffix automaton when `bisect` over sorted strings answers the prefix query in 3 lines.
- Storing parallel data in 4 dicts keyed the same way instead of one dict → dataclass values; four hash lookups where one suffices, and update-consistency bugs.
- Ignoring that `heapq` is a *min*-heap: for max behavior negate keys, and negate again on read — sign errors here produce plausible-looking wrong answers, so test with a 3-element example.
- Benchmarking with wall-clock on n = 100 and extrapolating: rehash spikes, GC, and cache effects don't extrapolate. Measure at target n.

## Verification / self-check

- Re-list the operation profile and confirm every operation used has the complexity you claimed *in the chosen library's implementation* (e.g., `SortedList.add` is O(n^0.5)-ish, not O(log n) — still usually fine, but claim it correctly).
- Check n against memory: node-based structures cost ~50–100 bytes/element in Python (object overhead); 10⁸ elements in a dict is not happening in 16GB — that's the sketch/bitset/numpy signal.
- For any hash-keyed design: are keys immutable, is iteration order relied upon, can an adversary choose keys?
- For amortized structures on a latency path: is a spike acceptable mid-operation?
- Trace one insert + one query by hand through the chosen structure at size 3 — most selection errors surface immediately.
