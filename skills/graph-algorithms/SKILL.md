---
name: graph-algorithms
description: Load when a problem involves networks, dependencies, states-and-moves, routing, matching, or flows — including problems where the graph is hidden. Covers traversal selection (BFS/Dijkstra/Bellman-Ford/0-1 BFS/A*), topological sort, SCC and 2-SAT, matching and flow reductions, MST reasoning, DAG DP, cycle detection, and representation choice.
---

# Graph Algorithms

The algorithms are strong-model baseline (0-1 BFS, Kahn, 2-SAT, project selection, low-link bridges, Eulerian conditions). The residual expert value is *seeing* the graph and running the routing checklist. This sheet keeps those.

## Modeling first (the actual skill)

- States are nodes, legal moves are edges, "minimum steps/cost to configuration" is shortest path — word ladders, water-jug, lock combinations, multi-entity positions (node = (pos_a, pos_b, turn)). When a problem resists, ask: what is a state, a move, the start/goal?
- Never materialize an implicit graph: BFS over a `neighbors(state)` function; bound |states| (product of component ranges) and cost the visited set before starting.
- Under-specified state is the same disease as bad DP state: keys collected, facing direction, fuel, parity — if the future depends on it, it's in the node. Constraints like "at most k tolls" or "alternate colors" multiply the state (node, budget) — Dijkstra/BFS run unchanged on the product graph at k·V cost.
- Many optimization problems are flow/matching in costume: assign-X-to-Y one-to-one → bipartite matching (rooks on allowed cells = rows×columns matching); "min removals to disconnect" → min-cut; "if you take A you must take B" with profits/costs → project-selection min-cut with ∞ dependency edges; König on bipartite: max independent set = V − max matching. Verify reductions in *both* directions — one-directional reductions produce feasible-but-suboptimal answers.

## Routing checklist (10 seconds, every time)

Weights: unweighted → BFS; {0,1} → 0-1 BFS deque; non-negative → Dijkstra; negative edges → Bellman-Ford (V-th-round improvement = negative cycle) or DAG relaxation; DAG → topo + relax (longest path trivial here, NP-hard generally); all-pairs V≤~400 → Floyd-Warshall with k **outermost**. Dijkstra on negative edges is silently wrong, not an error — grep the weights first. Directed vs undirected adjacency construction (both directions added exactly when intended) is the single most frequent input bug. Check disconnectedness (outer loop over unvisited starts), self-loops, parallel edges (they break parent-skip cycle checks and simple-graph assumptions).

## Implementation load-bearing lines (Python)

- BFS: mark visited **on enqueue** — on-dequeue re-enqueues duplicates, O(E) queue and blowup.
- Dijkstra without decrease-key: push duplicates, and on pop `if d > dist[u]: continue` — omitting it stays correct but degrades toward O(E·V); pushing `(node, dist)` instead of `(dist, node)` orders the heap by node id, a classic silent bug.
- 0-1 BFS: 0-edges `appendleft`, 1-edges `append`; deque holds at most two consecutive distance values — that invariant is why it's Dijkstra-correct without the heap.
- Recursion dies ~10³–10⁴ deep: iterative DFS/Tarjan for anything path-like (a line graph is the adversarial input); at V~10⁶ use int-indexed lists not dicts-of-tuples (5–10× and huge memory), intern states to ids at the boundary, read input via `sys.stdin.buffer`. `networkx` is fine to ≤10⁵ edges and one-shot calls; hand-roll beyond.
- A*: heuristic must never overestimate or you get confidently suboptimal paths that pass small tests; Manhattan only without diagonal moves.

## Pattern shortcuts most solutions miss

- Multi-source BFS: seed all sources at distance 0 — one line different from the per-source O(V·sources) TLE version.
- "Distance TO a target from every node" / "who can reach S" → one traversal on the reversed graph.
- Maximize-the-minimum edge on a path: binary search threshold + connectivity check, or single-shot Kruskal-style union until s,t connect; the max spanning tree contains the bottleneck path between every pair (MST does *not* contain shortest paths — those are different objectives entirely; but it *does* contain all minimax paths).
- Edge e in *some* MST ⟺ no path between its endpoints using only strictly lighter edges; second-best MST via per-non-tree-edge swap costs (w(e) − max on tree path). Cut property = exchange argument, reusable for "does the MST change if w(e) increases".
- Kahn gives cycle detection free (output < V nodes; leftovers are the tangled part = your error message), lexicographically-smallest order via min-heap, uniqueness ⟺ queue never holds two nodes.
- 2-SAT: clauses → implication edges (¬a→b, ¬b→a), UNSAT iff x and ¬x share an SCC, assignment = literal whose SCC is later in topo order; any "choose one of two per element + pairwise constraints" is 2-SAT in O(n + clauses) — capable engineers reach for backtracking here and time out.
- Min edges to make a digraph strongly connected = max(#source-SCCs, #sink-SCCs) of the condensation; already-strong special case → 0 (the formula would say 1).
- Cycle detection differs per graph type: directed needs three colors (gray→gray = cycle; visited-only false-positives on diamonds); undirected parent-skip must skip by *edge*, not vertex, when parallel edges exist; streaming undirected → DSU union-returns-False; functional graphs → Floyd.
- Bridges/articulation: Tarjan low-link one DFS (bridge iff low[v] > disc[u]); never remove-and-recheck at O(E·(V+E)).
- Eulerian (every EDGE once — Hamiltonian, every vertex, is the NP-hard one): undirected 0 or 2 odd-degree vertices; directed in=out except one ±1 pair; build with Hierholzer. Triggers: dominoes, word-overlap chains, itinerary reconstruction.
- Grid problems: the grid IS the adjacency; `for dr, dc in ((0,1),(0,-1),(1,0),(-1,0))` is the edge iterator; walls-cost-1/free-cost-0 problems are 0-1 BFS in disguise, not obstacle pathfinding.
- Flow calibration: `networkx.maximum_flow` to ~10⁴–10⁵ edges; Dinic's for contest scale (O(E√V) on unit/bipartite); dense min-cost matching → `linear_sum_assignment` (1000×1000 well under a second) — don't hand-roll.
- DAG counting can be exponential in value — bigints or take the modulus; BFS distance counts *edges*, nodes-on-path = dist+1 — re-read what the problem counts; path reconstruction needs parents stored during traversal, distances alone can't recover it.

## Verification / self-check

- Re-check the three routing questions (negatives? zeros? DAG?) after reading the actual data, not the problem statement.
- Test on: disconnected input, single node, self-loop, parallel edges, and a depth->10⁴ path graph (recursion probe).
- For reductions: map a solution back and forth explicitly; sanity-check units (edges vs nodes, cost vs count).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed every probe: weight-profile routing with the Dijkstra-negative and Floyd-k-order pitfalls, visited-on-enqueue, the stale-skip line and tuple-order bug, Kahn cycle/uniqueness/lex tricks, full 2-SAT with assignment extraction, max(sources,sinks) with the already-strong special case, three-color vs parent-skip vs DSU cycle detection, 0-1 BFS with the deque invariant, project-selection reduction, low-link rule, multi-source/reverse-graph/bottleneck patterns, MST-vs-shortest-path (plus the minimax-path property, now added here), Eulerian conditions/Hierholzer, and the Python-at-scale notes.
- No substantive gaps; retained value is the modeling reflex (state/move/product-graph), the 10-second routing checklist, and pattern completeness.
