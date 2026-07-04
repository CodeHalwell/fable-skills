---
name: graph-algorithms
description: Load when a problem involves networks, dependencies, states-and-moves, routing, matching, or flows — including problems where the graph is hidden. Covers traversal selection (BFS/Dijkstra/Bellman-Ford/0-1 BFS/A*), topological sort, SCC and 2-SAT, matching and flow reductions, MST reasoning, DAG DP, cycle detection, and representation choice.
---

# Graph Algorithms

## Core mental model

- **Modeling is the hard part; the algorithms are commodities.** Expert value is in *seeing* the graph: states are nodes, legal moves are edges, and "minimum steps/cost to reach a configuration" is shortest path. Word ladders, water-jug puzzles, lock combinations, "minimum operations to transform X into Y", multi-entity positions (cat-and-mouse: node = (pos_a, pos_b, turn)) — all shortest-path on implicit graphs. When a problem resists direct attack, ask: *what is a state, what is a move, what is the start/goal?*
- **Never materialize an implicit graph if you can generate neighbors on the fly.** BFS over `neighbors(state)` as a function; enumerating all edges of a 10⁶-state space up front wastes memory and often exceeds it.
- **Edge weights select the traversal — this is a lookup, not a decision.** Unweighted → BFS. Weights {0,1} → 0-1 BFS (deque). Non-negative → Dijkstra. Negative edges → Bellman-Ford (or DAG relaxation if acyclic). Negative *cycles reachable on your path* → shortest path is undefined; detect and report. Using Dijkstra with negative edges is a silent-wrong-answer, not an error.
- **Directionality and structure gate the toolset.** DAG unlocks: topological sort, linear-time shortest/longest path, DP over nodes. Undirected unlocks: union-find connectivity, MST. General digraphs need SCC condensation to *become* a DAG first. Ask "is it a DAG? can I make it one?" early.
- **Many optimization problems are flow/cut/matching in costume.** "Assign workers to jobs" = bipartite matching. "Minimum capacity to sever A from B" = min-cut. Learning to smell these reductions is worth more than knowing Dinic's internals — libraries (`networkx`, `scipy`) implement the algorithms.

## Traversal selection table

| Situation | Algorithm | Critical requirement / pitfall |
|---|---|---|
| Fewest edges/moves, unweighted | BFS | Mark visited **on enqueue**, not on dequeue — else the queue blows up with duplicates (exponential in bad cases) |
| Weights all 0 or 1 | 0-1 BFS: deque, 0-edges `appendleft`, 1-edges `append` | O(V+E), beats Dijkstra's log factor; node may be popped twice — skip if already finalized |
| Non-negative weights | Dijkstra (heap + lazy deletion) | NEVER with negative edges — its greedy finalization assumes distances only grow along paths |
| Any weights, or need negative-cycle detection | Bellman-Ford (V−1 rounds; a V-th improvement ⇒ negative cycle) | O(VE); SPFA variant is fast on average, adversarially quadratic |
| Any weights on a DAG | Topo order + relax | O(V+E), handles negative fine; also gives *longest* path (NP-hard on general graphs, trivial on DAGs) |
| All-pairs, V ≤ ~400 | Floyd-Warshall | Loop order MUST be `for k: for i: for j` — k outermost; swapping it is the classic silent bug |
| Single goal, good distance estimate | A* | Heuristic must be **admissible** (never overestimates) for optimality, and consistent for efficient re-expansion; h = 0 degrades to Dijkstra; Euclidean/Manhattan distance are the standard safe choices on grids (Manhattan only if no diagonal moves) |
| Both-ends known, big branching factor | Bidirectional BFS | Meet-in-middle: 2·b^(d/2) ≪ b^d; the answer check happens when frontiers *touch*, not when goal is dequeued |

Dijkstra idiom (Python) — lazy deletion, no decrease-key:
```python
import heapq
def dijkstra(adj, src):                    # adj[u] = [(v, w), ...]
    dist = {src: 0}
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist.get(u, float('inf')):  # stale entry — the load-bearing line
            continue
        for v, w in adj[u]:
            nd = d + w
            if nd < dist.get(v, float('inf')):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist
```
Omitting the stale-skip keeps the answer correct but degrades performance to O(E·V) on dense inputs. Storing `(node, dist)` instead of `(dist, node)` in the tuple orders the heap by node id — a classic silent bug.

## Topological sort — the dependency workhorse

Triggers: build orders, course prerequisites, "must happen before", spreadsheet/formula evaluation, task scheduling, resolving imports. Kahn's algorithm (in-degree + queue) is preferred over DFS post-order because it detects cycles for free: **if the output has fewer than V nodes, a cycle exists** — and the leftover nodes with nonzero in-degree are exactly the cycle-tangled part, which is your error message. For "lexicographically smallest valid order", swap the queue for a min-heap. For "is the order *unique*": yes iff the queue never holds two nodes at once (equivalently, the topo order forms a Hamiltonian path).

## SCC and 2-SAT

- **SCC condensation** turns any digraph into a DAG of components, unlocking DP. Triggers: "mutually reachable", "collapse cyclic dependencies", "minimum edges to make strongly connected" (answer: max(#source-components, #sink-components) of the condensation, special-case a single SCC → 0). Use Tarjan's (one DFS) or `networkx.strongly_connected_components`. In Python, recursion depth kills naive Tarjan at V ≳ 10⁴ — use an iterative version or the library.
- **2-SAT** = SCC applied. Each clause (a ∨ b) becomes implications ¬a→b and ¬b→a in a graph over 2n literal-nodes. Satisfiable iff no variable shares an SCC with its negation. To extract an assignment: a variable is TRUE iff its positive literal's SCC comes *after* its negation's in reverse topological order. Triggers: binary choices with pairwise constraints — "each item placed left or right such that no two conflicting items...", radio-frequency pairs, seating with mutual-exclusion. Any "choose one of two per element + pairwise implications" is 2-SAT, solvable in O(n + clauses) — capable engineers reach for backtracking here and time out.

## Matching and flow reductions

- **Bipartite matching triggers:** assign X to Y, one-to-one, with compatibility constraints — workers/jobs, students/schools, rows/columns (placing non-attacking rooks on allowed cells = matching rows to columns!). Use Hopcroft-Karp (`networkx.bipartite.maximum_matching`) or reduce to max-flow with unit capacities.
- **König's theorem** (bipartite only): max matching = min vertex cover, and max independent set = V − max matching. Problems asking "minimum guards to watch all edges" or "maximum conflict-free subset" on bipartite structure are matching problems in costume.
- **Max-flow min-cut applications — the reduction patterns:**
  - *Image segmentation / binary labeling:* source = "foreground", sink = "background"; per-pixel edges with label-affinity capacities; neighbor edges with smoothness penalty. Min-cut = optimal labeling (this is exact for submodular pairwise energies).
  - *Project selection / open-pit mining:* profits from source, costs to sink, dependencies as ∞ edges; answer = total profit − min cut. The ∞ edges encode "if you take A you must take B" — a hugely reusable trick.
  - *Scheduling with capacities:* jobs → machines/time-slots, capacities model limits; feasibility = whether flow saturates all job edges.
  - *Vertex capacities / node-disjoint paths:* split each node v into v_in → v_out with the capacity on that internal edge.
  - *Edge-disjoint paths* between s and t = max-flow with unit edge capacities (Menger's theorem).
- Calibration: `networkx.maximum_flow` (default preflow-push) handles ~10⁴–10⁵ edge graphs comfortably; for contest-scale performance write Dinic's (O(E√V) on unit-capacity/bipartite graphs). Min-cost matching on a dense cost matrix: `scipy.optimize.linear_sum_assignment` (Hungarian, handles 1000×1000 in well under a second) — don't hand-roll.

## MST — exchange-property reasoning

The **cut property** is the reusable tool, not the algorithms: for any cut, the lightest edge crossing it is in *some* MST; with distinct weights, in *the* MST. Its proof is an exchange argument (swap the lightest crossing edge into any spanning tree that omits it; the cycle created must contain another crossing edge at least as heavy — remove it). Use this reasoning directly for questions like: "is edge e in some MST?" (yes iff e is a minimum-weight edge across some cut — check: no path between its endpoints using only strictly lighter edges), "does the MST change if edge e's weight increases?", "second-best MST" (for each non-tree edge, the swap cost is w(e) − max-weight-on-tree-path). Kruskal (sort edges + union-find) for sparse/edge-list inputs; Prim with a heap for dense/adjacency inputs. Maximum spanning tree = negate weights. **MST does NOT give shortest paths** — the s→t path in an MST can be arbitrarily longer than the shortest path; conflating these is a real and common error.

## DAG dynamic programming

On a DAG, "DP" and "graph traversal" merge: any recurrence over nodes evaluated in topological order. Longest path (critical path/scheduling), counting s→t paths, minimum path cover (via matching), reachability with constraints. Two idioms: (1) explicit topo order + loop; (2) memoized DFS (`@lru_cache` over node) — cleaner, but recursion depth. Any "count the ways / best value moving only forward" (grid moving right/down, version-upgrade chains, poset chains) is DAG DP even when no graph is drawn. Counting paths can be exponential in value — use Python bigints or apply the required modulus.

## Cycle detection — per graph type (they differ!)

- **Directed:** DFS with three colors (white/gray/black); a gray→gray edge is a cycle. OR Kahn's: leftover nodes = cycle. Two-color visited is NOT enough for directed graphs — a black node reached again is fine (diamond), only gray means cycle. This is the exact bug in most wrong directed-cycle detectors.
- **Undirected (DFS):** an edge to a visited vertex that is not the immediate parent = cycle. The parent check must be by *edge*, not vertex, if multi-edges are possible (two parallel edges ARE a cycle).
- **Undirected (streaming edges):** union-find; `union` returning "already connected" = cycle. This is the right tool during Kruskal or edge-by-edge input.
- **Functional graphs (each node one out-edge):** Floyd's tortoise-and-hare or just walk with a visited set; cycle entry point via the standard two-phase Floyd.

## Representation choice by density

- Adjacency **list** (`defaultdict(list)` or list-of-lists): the default; O(V+E) memory, right for E ≪ V².
- Adjacency **matrix**: dense graphs (E ≈ V²), V ≤ ~5000; O(1) edge queries; required by Floyd-Warshall; bitset rows (`int` bitmasks in Python) accelerate transitive closure by 64×.
- Edge list: Kruskal, Bellman-Ford — algorithms that just iterate all edges.
- Implicit `neighbors(state)` function: state-space searches; add a `dist`/`visited` dict keyed by state (make states hashable: tuples, frozensets, or serialize grids as `tuple(map(tuple, grid))` or a bytes string).
- Python performance notes: for V ~ 10⁶ use lists indexed by int ids, not dicts of tuples (5–10× and huge memory difference); intern states to ids at the boundary. Recursion-based DFS dies around depth 10³–10⁴ — convert to an explicit stack for anything path-like (a line graph is the adversarial input that finds this bug).

## Failure modes & pitfalls (cross-cutting)

- Running Dijkstra with negative edges — silently wrong, not an exception. Grep the weights before choosing.
- BFS marking visited on dequeue instead of enqueue → duplicate expansion, TLE/memory blow-up.
- Forgetting the graph may be **disconnected**: connected-component counts, MST ("forest?"), and "visit all nodes" traversals need the `for each unvisited start` outer loop.
- Treating a grid problem as needing a graph library: the grid IS the adjacency structure; `for dr, dc in ((0,1),(0,-1),(1,0),(-1,0)):` is the edge iterator. Bounds-check before visiting; do not mutate the grid as your visited set if the input must survive.
- Self-loops and parallel edges breaking assumptions: parent-skip cycle detection, matrix representation (loses multi-edges), flow reductions (usually fine, but dedupe if the algorithm assumes simple).
- State-space search where the state is under-specified — same disease as DP state design: if the future depends on it, it's in the node. (Keys collected, direction facing, fuel remaining, parity of steps.)
- A* with an inadmissible heuristic (e.g., weighted h to "speed up") returning suboptimal paths while looking correct on small tests. If optimality is required, prove h never overestimates.
- Off-by-one on "path length": BFS distance counts *edges*; "number of nodes on the path" is dist + 1. Re-read what the problem counts.
- Using `networkx` inside hot loops at scale — it stores dict-of-dict; building it for 10⁶ edges costs seconds and hundreds of MB. Fine for ≤10⁵ edges and one-shot algorithm calls; hand-roll beyond.

## Worked micro-example: hidden graph + 0-1 BFS

"Minimum walls to break to get from (0,0) to (n−1,m−1) on a grid" — not pathfinding-with-obstacles but a *weighted* problem in disguise: moving into a free cell costs 0, into a wall costs 1. 0-1 BFS:

```python
from collections import deque
def min_walls(grid):                     # grid[r][c] in {0 free, 1 wall}
    n, m = len(grid), len(grid[0])
    INF = float('inf')
    dist = [[INF] * m for _ in range(n)]
    dist[0][0] = grid[0][0]
    dq = deque([(0, 0)])
    while dq:
        r, c = dq.popleft()
        for dr, dc in ((0,1),(0,-1),(1,0),(-1,0)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < n and 0 <= nc < m:
                nd = dist[r][c] + grid[nr][nc]
                if nd < dist[nr][nc]:
                    dist[nr][nc] = nd
                    if grid[nr][nc]:  dq.append((nr, nc))      # weight 1 → back
                    else:             dq.appendleft((nr, nc))  # weight 0 → front
    return dist[n-1][m-1]
```
The deque invariant: distances in the deque are non-decreasing and span at most two consecutive values — that is why front-insertion of 0-edges preserves Dijkstra's correctness without a heap. Recognizing "{0,1} costs" saved a log factor and a heap import.

## Verification / self-check

- Confirm the traversal matches the weight profile (re-check for negative weights, 0-weights, and whether the graph is a DAG — three questions, ten seconds).
- Directed vs undirected: does the adjacency construction add both `u→v` and `v→u` exactly when intended? (The single most frequent graph-input bug.)
- Test on: disconnected input, single node, self-loop, two nodes with parallel edges, and a path graph of depth > 10⁴ (recursion check).
- For reductions (flow/matching/2-SAT): verify both directions — a solution to the reduced problem maps back to a valid original solution, AND every original solution corresponds to a reduced one. One-directional reductions produce feasible-but-suboptimal answers.
- For implicit graphs: bound |states| explicitly (product of component ranges) and confirm memory for the visited set at that bound.
- Sanity-check the answer's units: edges vs nodes, cost vs count, and whether the problem wants the path itself (store parents during traversal — reconstructing after the fact is not possible from distances alone).
