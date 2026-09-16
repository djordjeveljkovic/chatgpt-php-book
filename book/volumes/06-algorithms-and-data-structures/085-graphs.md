---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 85
title: Graphs
slug: graphs
status: complete
summary: ../../_ai/chapter-summaries/085-graphs-summary.md
---

# Chapter 85 — Graphs

## Why This Matters

Many backend problems are relationships between things rather than lists or hierarchies. A route planner connects places by roads. A dependency checker connects packages by requirements. A permission system connects users, roles, and resources. A social feature connects accounts that follow one another. These models are graphs, whether or not the application calls them that.

The right model changes the questions we can answer. Is one account reachable from another? Which prerequisites must be processed first? Is there a cycle? What is the fewest number of transitions between two states? A graph makes vertices and edges explicit, then lets a traversal answer questions without repeatedly scanning every record.

PHP can represent a modest graph in arrays and objects, but representation has a real memory cost. A graph with millions of edges may belong in a database, a graph service, or a purpose-built storage layer rather than a PHP-FPM worker's heap. This chapter develops the model and its algorithms, then draws that boundary.

## Mental Model

A graph is a set of **vertices** (also called nodes) and **edges** (relationships between vertices):

```text
Harbor ─── Market ─── Museum
   │                     │
   └────── Station ──────┘
```

An edge can have a direction, a weight, or both. In an undirected graph, `A — B` means the relationship works both ways. In a directed graph, `A → B` means the relationship goes from A to B; it does not imply `B → A`. A weight stores a property such as distance, duration, cost, or capacity. The meaning of the weight is domain policy, not something the graph decides.

A **path** is a sequence of vertices connected by edges. A **cycle** is a path that starts and ends at the same vertex and contains at least one edge. A graph may be disconnected: some vertices have no path to others. Unlike a strict tree (Chapter 82), a graph may have multiple paths to one vertex, cycles, self-edges, and several connected components.

## Core Concept

The model begins with two questions: what counts as a vertex, and what exactly does an edge mean? “User A is connected to User B” might mean a mutual friendship, a one-way follow, or a permission inheritance link. The choice changes whether edges are directed and whether a traversal may move in both directions.

Common graph properties include:

- **Directed or undirected:** whether an edge has a direction.
- **Weighted or unweighted:** whether an edge carries a cost or other value. “Unweighted” algorithms can be understood as treating every edge as equal cost.
- **Simple graph or multigraph:** whether self-edges and repeated edges are allowed.
- **Cyclic or acyclic:** whether a path can return to its starting vertex. A directed acyclic graph (DAG) is useful for prerequisite and build order, but a directed graph with a cycle cannot be topologically ordered as a whole.
- **Connected or disconnected:** for an undirected graph, whether every vertex is reachable from every other vertex. For directed graphs, distinguish reachability from strong connectivity, where every vertex can reach every other one.

These properties are part of the contract. Inserting an undirected edge usually stores a neighbor in both directions. A directed edge stores only its outgoing relationship. A weighted edge keeps its weight with that relationship. Decide whether repeated edges are meaningful or should be rejected/deduplicated; otherwise the same relationship can be counted or traversed more than once.

## Representing a Graph

### Adjacency list

An adjacency list stores each vertex and its outgoing neighbors. Here is the useful shape for a weighted graph:

```text
harbor  → [(market, 4), (station, 12)]
market  → [(harbor, 4), (museum, 3)]
station → [(harbor, 12), (museum, 2)]
museum  → [(market, 3), (station, 2)]
```

The representation stores one list per vertex and one record per directed adjacency. An undirected edge appears twice, once at each endpoint, except a self-edge which appears once in this chapter's implementation. Repeated calls to add the same edge are retained as parallel edges.

For `V` vertices and `E` edges, an adjacency list takes O(V + E) space. Iterating a vertex's neighbors costs O(degree(v)); traversing the whole graph costs O(V + E). Finding whether one particular edge exists by scanning a vertex's neighbor list costs O(degree(v)); a second set or index can make that lookup faster at a memory and maintenance cost.

This is usually the natural representation for sparse graphs, where each vertex connects to a small fraction of all vertices. PHP arrays are ordered maps, with per-entry overhead (Chapter 75), so O(V + E) still may mean substantial memory. If vertex IDs are external strings, define canonical identity. This example prefixes internal array keys so IDs like `"17"` cannot be silently converted to integer keys under PHP's array-key rules.

### Adjacency matrix

An adjacency matrix assigns every vertex an integer position and stores an entry for every possible pair. A boolean matrix answers whether there is an edge; a weighted matrix can store the weight, with a separate sentinel for “no edge” so that a zero-weight edge remains representable.

A matrix needs O(V²) space whether the graph is sparse or dense. Edge existence is O(1), and enumerating one vertex's neighbors costs O(V). A matrix is reasonable for a small, dense graph or when checking arbitrary pairs dominates. For a large sparse graph, it spends memory on nonexistent edges. In PHP, a nested array matrix adds substantial hash-table overhead on top of the O(V²) cells; a packed string or specialized extension may be more compact for large fixed matrices.

### Edge list

An edge list stores records such as `[from, to, weight]`. It is compact when the main work is processing every edge once, and useful as an import/export shape. Looking up all neighbors of one vertex requires a scan unless the list is indexed into an adjacency structure. A database table with `from_id`, `to_id`, and optional `weight` is an edge list backed by indexes chosen for the queries the application runs.

No representation wins for every operation. Select it from the graph's density, size, mutation rate, and query pattern.

## A PHP Adjacency-List Model

This small class represents directed or undirected graphs with integer edge weights. It allows negative weights as data; algorithms that interpret weights as costs must state their own restrictions. It retains duplicate edges and treats IDs as exact strings.

```php
<?php

declare(strict_types=1);

function graphArrayKey(string $id): string
{
    // A non-numeric prefix avoids PHP's conversion of integer-looking string keys.
    return 'vertex:' . $id;
}

final class Graph
{
    /** @var array<string, string> Internal key => original vertex ID. */
    private array $vertices = [];

    /** @var array<string, list<array{to: string, weight: int}>> */
    private array $adjacency = [];

    public function __construct(private bool $directed)
    {
    }

    public function isDirected(): bool
    {
        return $this->directed;
    }

    public function addVertex(string $id): void
    {
        $key = graphArrayKey($id);
        if (array_key_exists($key, $this->vertices)) {
            return;
        }

        $this->vertices[$key] = $id;
        $this->adjacency[$key] = [];
    }

    public function addEdge(string $from, string $to, int $weight = 1): void
    {
        $this->addVertex($from);
        $this->addVertex($to);

        $this->adjacency[graphArrayKey($from)][] = [
            'to' => $to,
            'weight' => $weight,
        ];

        if (!$this->directed && $from !== $to) {
            $this->adjacency[graphArrayKey($to)][] = [
                'to' => $from,
                'weight' => $weight,
            ];
        }
    }

    public function hasVertex(string $id): bool
    {
        return array_key_exists(graphArrayKey($id), $this->vertices);
    }

    /** @return list<string> */
    public function vertices(): array
    {
        return array_values($this->vertices);
    }

    /** @return list<array{to: string, weight: int}> */
    public function neighbors(string $id): array
    {
        $key = graphArrayKey($id);
        if (!array_key_exists($key, $this->adjacency)) {
            throw new InvalidArgumentException("Unknown vertex: {$id}");
        }

        return $this->adjacency[$key];
    }
}
```

The class keeps exact IDs in `$vertices` and uses prefixed keys only for PHP array lookup. This also means the IDs `"17"` and `"017"` remain distinct. The adjacency array is mutable and local to the PHP process; it is not a database transaction or a shared graph service. This is a teaching representation, not a claim that hand-written graph storage is better than an existing library or database.

## Traversal and Shortest Paths by Edge Count

Breadth-first search (BFS) uses a FIFO queue to visit vertices in layers. Chapter 78 introduced the basic queue traversal; here we use the graph model to answer a more specific question: find a shortest path measured by **number of edges** and reconstruct that path. BFS is appropriate because every edge counts equally for this metric, even if the graph happens to carry weights for another purpose.

The algorithm records each vertex's predecessor the first time it is discovered. It marks the vertex as visited when enqueued, so cycles and repeated edges cannot add it indefinitely. Once the target is reached, following predecessor links backwards reconstructs a shortest path.

```php
/**
 * Returns a path with the fewest edges, or null if either endpoint is missing
 * or the target is unreachable from the start.
 *
 * @return list<string>|null
 */
function shortestPathByEdgeCount(Graph $graph, string $start, string $target): ?array
{
    if (!$graph->hasVertex($start) || !$graph->hasVertex($target)) {
        return null;
    }

    $startKey = graphArrayKey($start);
    $targetKey = graphArrayKey($target);
    $visited = [$startKey => true];
    /** @var array<string, string> $parent */
    $parent = [];
    $queue = new SplQueue();
    $queue->enqueue($start);

    while (!$queue->isEmpty()) {
        $current = $queue->dequeue();
        if ($current === $target) {
            break;
        }

        foreach ($graph->neighbors($current) as $edge) {
            $neighbor = $edge['to'];
            $neighborKey = graphArrayKey($neighbor);
            if (isset($visited[$neighborKey])) {
                continue;
            }

            $visited[$neighborKey] = true; // Mark on enqueue, not on dequeue.
            $parent[$neighborKey] = $current;
            $queue->enqueue($neighbor);
        }
    }

    if (!isset($visited[$targetKey])) {
        return null;
    }

    $path = [];
    $current = $target;
    while (true) {
        $path[] = $current;
        if ($current === $start) {
            break;
        }
        $current = $parent[graphArrayKey($current)];
    }

    return array_reverse($path);
}
```

A route graph can exercise the distinction between fewest edges and lowest total cost:

```php
$routes = new Graph(directed: false);
$routes->addEdge('harbor', 'market');
$routes->addEdge('market', 'museum');
$routes->addEdge('harbor', 'station');
$routes->addEdge('station', 'museum');
$routes->addEdge('harbor', 'old-town');
$routes->addEdge('old-town', 'gallery');
$routes->addEdge('gallery', 'museum');

$path = shortestPathByEdgeCount($routes, 'harbor', 'museum');
// One shortest answer is ['harbor', 'market', 'museum'];
// another equal-length answer may be returned if neighbor order changes.
```

The result depends on adjacency iteration order only when multiple shortest paths tie. Do not rely on that choice unless the API defines a deterministic tie policy. If each edge instead represents a travel time and the requirement is minimum total time, this BFS is the wrong algorithm: it minimizes edge count and ignores `weight`. Use a shortest-path algorithm whose assumptions match the weights, such as Dijkstra for nonnegative costs; negative weights require a different method and careful treatment of negative cycles.

For `Vᵣ` vertices and `Eᵣ` edges reachable from the start, BFS takes O(Vᵣ + Eᵣ) expected time with adjacency-list traversal and O(Vᵣ) additional space for the queue, visited set, and predecessor map. It enqueues a vertex at most once. An adjacency matrix would make a full neighbor scan O(V) per visited vertex, for O(Vᵣ · V) traversal time.

## Cycles and Depth-First Traversal

A `visited` set prevents a traversal from revisiting a vertex forever. It does not by itself tell whether a directed graph contains a cycle: an edge to a vertex visited earlier may point to a finished branch, not an ancestor. For directed cycle detection, use three states:

- unseen: no traversal has entered the vertex;
- active: the current depth-first path is exploring it;
- finished: all outgoing edges have been examined.

An edge to an active vertex is a back edge and proves a cycle. The iterative implementation below stores each frame's next-neighbor position instead of using PHP recursion, so a long chain does not consume one PHP call frame per vertex.

```php
function hasDirectedCycle(Graph $graph): bool
{
    if (!$graph->isDirected()) {
        throw new InvalidArgumentException('Directed cycle detection requires a directed graph.');
    }

    /** @var array<string, int> $state 1 = active, 2 = finished; missing = unseen. */
    $state = [];

    foreach ($graph->vertices() as $start) {
        $startKey = graphArrayKey($start);
        if (isset($state[$startKey])) {
            continue;
        }

        $state[$startKey] = 1;
        /** @var list<array{node: string, next: int}> $stack */
        $stack = [['node' => $start, 'next' => 0]];

        while ($stack !== []) {
            $top = count($stack) - 1;
            $node = $stack[$top]['node'];
            $neighbors = $graph->neighbors($node);

            if ($stack[$top]['next'] >= count($neighbors)) {
                $state[graphArrayKey($node)] = 2;
                array_pop($stack);
                continue;
            }

            $edge = $neighbors[$stack[$top]['next']];
            ++$stack[$top]['next'];
            $neighborKey = graphArrayKey($edge['to']);
            $neighborState = $state[$neighborKey] ?? 0;

            if ($neighborState === 1) {
                return true;
            }
            if ($neighborState === 0) {
                $state[$neighborKey] = 1;
                $stack[] = ['node' => $edge['to'], 'next' => 0];
            }
        }
    }

    return false;
}
```

Starting a DFS from every still-unseen vertex handles disconnected components. The state map prevents a completed component from being explored twice. This takes O(V + E) expected time and O(V) additional space with an adjacency list. For an undirected graph, cycle detection must also remember the parent edge (and account for parallel edges if they are allowed); simply treating every edge to a visited neighbor as a cycle would incorrectly flag the edge back to a vertex's parent.

The directed test is a useful way to check both edge direction and disconnected components:

```php
$dependencies = new Graph(directed: true);
$dependencies->addEdge('application', 'library');
$dependencies->addEdge('library', 'runtime');
$dependencies->addVertex('unrelated');

var_dump(hasDirectedCycle($dependencies)); // false
$dependencies->addEdge('runtime', 'application');
var_dump(hasDirectedCycle($dependencies)); // true
```

A cycle in a dependency graph usually means “no valid order exists,” not that the traversal implementation failed. The next step is to report a useful cycle path or identify the domain records that created it. A boolean result is enough for validation but often not enough for an operator to repair bad data.

## What PHP Does

The graph in this chapter uses PHP arrays as maps and lists. PHP accepts integer and string array keys and casts certain integer-looking string keys to integers; prefixing an external ID makes its internal key unambiguous. Neighbor values remain regular PHP strings and integers. `SplQueue` provides the FIFO operations used by BFS, while the graph itself is ordinary application data.

The asymptotic costs describe the adjacency-list algorithm, not a language guarantee about PHP array memory or elapsed time. `SplQueue` and array choices have different allocation and per-entry costs. Copy-on-write and object allocation may affect peak memory and runtime, especially when graphs are built from large query results; Chapters 49–52 and 74 explain those PHP memory behaviors.

## Workload Boundaries

An in-memory graph is useful when the relevant vertices and edges fit within a measured memory budget and an algorithm must inspect many relationships in one process. Examples include validating a small package manifest, traversing a bounded authorization graph, calculating routes over a cached regional network, or finding affected components in an import batch.

A graph stored in relational tables can use indexes and recursive queries for some traversals; a graph database may suit relationship-heavy, multi-hop query patterns. Neither storage choice removes the need to define direction, depth limits, consistency, or result-size bounds. Loading a high-degree or unbounded relationship set into PHP can still create excessive memory use or work. A network of services also cannot be traversed as if remote calls were free: latency, timeouts, stale responses, partial failure, and authorization apply at every boundary.

Graph consistency is temporal. If follow relationships or dependencies change while an algorithm runs, the answer is based on the edges the process observed. Production code should decide whether it needs a stable database snapshot, a versioned graph, eventual consistency, or a retry when the source changes. A local traversal does not make the underlying records atomic.

For user-controlled graph input, bound the number of vertices, edges, and maximum traversal depth. Cycles are normal graph structure but can cause runaway recursion in careless code. Avoid recursive DFS on untrusted, very deep graphs; use an explicit stack, as above. Validate that a caller may access each vertex or edge before revealing reachability, since “can reach resource X” may itself disclose protected relationships.

## Testing

Test graph behavior at the boundaries that distinguish it from a tree or list:

1. Add vertices with IDs `"17"` and `"017"`; verify they remain separate and can each have edges.
2. Verify a directed edge is traversable only in its declared direction and an undirected edge appears from both endpoints.
3. Check isolated vertices, missing endpoints, self-edges, repeated edges, and disconnected components.
4. For BFS, test start equal to target, an unreachable target, a cycle, repeated edges, and multiple equal-length shortest paths. Assert path length and valid adjacency rather than an arbitrary tie result.
5. Give two routes different weights and verify that `shortestPathByEdgeCount()` chooses by edge count, not total weight.
6. For directed-cycle detection, test a chain, a back edge, a self-edge, a cross-edge to a finished branch, and a cycle in a component disconnected from the first vertex.
7. For any weighted shortest-path implementation, test zero weights, negative-weight policy, disconnected vertices, and overflow/range behavior.
8. Compare traversal results against a simple reference implementation on generated small graphs. Property tests can assert that every consecutive pair in a returned path has an edge and that a returned unweighted shortest path has no longer path in the reference graph.

Use deterministic fixtures for unit tests and bounded random graphs for property tests. Avoid timing assertions: test the operation count or use a separate benchmark with representative graph density and ID lengths.

## Common Mistakes

- Treating an edge as symmetric when the domain relationship is directed.
- Using BFS to minimize total weighted cost; BFS minimizes edge count when all transitions are treated equally.
- Marking vertices visited only after dequeue, allowing cycles and duplicate edges to enqueue the same vertex many times.
- Assuming a visited set alone proves that a directed graph contains a cycle.
- Applying tree assumptions to a graph that can have several parents or disconnected components.
- Using an adjacency matrix for a large sparse graph without accounting for O(V²) space.
- Assuming a PHP adjacency list is memory-cheap just because its abstract space is O(V + E).
- Trusting a graph traversal over changing records to represent one consistent snapshot.
- Loading an unbounded remote or database graph into one PHP process.
- Exposing reachability answers without checking authorization.

## Senior Engineer Thinking

Start with the domain invariant: what is a vertex, what does a directed edge mean, can edges repeat, and can weights be negative? Then list the queries and their frequency. If the workload is “iterate every relationship once,” an edge list may be enough. If it is “walk neighbors repeatedly in a sparse graph,” an adjacency list is natural. If it is “check any pair in a small dense graph,” a matrix may be simpler.

State exactly what an algorithm optimizes. “Shortest path” is incomplete until the cost is defined: fewest edges, least distance, earliest arrival, or another objective. Then account for representation cost, PHP memory, source-data consistency, and the trust boundary. If the graph outgrows a single process, move the storage/traversal boundary deliberately instead of trying to fix the problem only by tuning a PHP array.

## Exercises

1. Model package dependencies as a directed graph. Define how duplicate requirements and a cycle should be reported, then implement a topological-order validation for a small manifest.
2. Extend `Graph` with `removeEdge()`. State whether it removes one matching parallel edge or all matches, and test both directed and undirected behavior.
3. Add an undirected connected-components function using an explicit stack or queue. Include isolated vertices and a self-edge.
4. Create a small weighted route graph where the fewest-edge path is not the cheapest path. Explain why BFS gives the wrong answer for the cost requirement and identify the assumptions a replacement algorithm needs.
5. Compare adjacency-list and adjacency-matrix storage for 2,000 vertices at 1% and 80% density. Estimate abstract entries before measuring actual PHP memory.
6. Design a database-backed reachability query with a maximum depth. Describe which indexes, snapshot semantics, result cap, and authorization checks it needs.

## Review Questions

1. What distinguishes a vertex from an edge, and what changes when the graph is directed?
2. How does an adjacency list trade space and neighbor iteration against an adjacency matrix?
3. Why can PHP string IDs that resemble integers need canonical internal keys?
4. What does BFS guarantee when each edge has equal cost? What does it not guarantee for weighted routes?
5. Why should a BFS mark a vertex visited when it is enqueued?
6. Why does directed cycle detection distinguish active vertices from finished vertices?
7. What additional information is needed to detect cycles correctly in an undirected graph?
8. Why can a graph query over mutable records return an answer that never existed as one consistent snapshot?
9. Which graph workloads should move out of a PHP process, and what query limits remain necessary after that move?

## Summary

A graph models vertices and their relationships, which may be directed, undirected, weighted, disconnected, and cyclic. An adjacency list uses O(V + E) space and visits sparse neighborhoods efficiently; an adjacency matrix uses O(V²) space but checks a pair in O(1); an edge list is convenient for bulk edge processing. PHP arrays can model these structures, but their runtime memory cost must be measured.

BFS with a visited set and predecessor map finds a path with the fewest edges, not the lowest sum of edge weights. Directed cycle detection needs traversal state that distinguishes active ancestors from finished vertices. Before implementing a graph algorithm, define vertex identity, edge semantics, weight rules, consistency, and result bounds. For large or mutable graphs, choose an indexed database or graph service based on the actual queries and keep authorization, depth, latency, and memory limits explicit.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: `SplQueue`](https://www.php.net/manual/en/class.splqueue.php)
- [PHP Manual: `SplQueue::enqueue()`](https://www.php.net/manual/en/splqueue.enqueue.php)
- [PHP Manual: `SplQueue::dequeue()`](https://www.php.net/manual/en/splqueue.dequeue.php)
- [Chapter 49 — HashTables](../../volumes/04-php-under-the-hood/049-hashtables.md)
- [Chapter 52 — Copy-on-Write](../../volumes/04-php-under-the-hood/052-copy-on-write.md)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 75 — Arrays and Hash Maps](./075-arrays-and-hash-maps.md)
- [Chapter 76 — Sets](./076-sets.md)
- [Chapter 78 — Queues](./078-queues.md)
- [Chapter 82 — Trees](./082-trees.md)
- [Chapter 86 — Intervals](./086-intervals.md)
