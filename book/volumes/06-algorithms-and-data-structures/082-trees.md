---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 82
title: Trees
slug: trees
status: complete
summary: ../../_ai/chapter-summaries/082-trees-summary.md
---

# Chapter 82 — Trees

## Why This Matters

Many application values are not flat lists. A product category contains subcategories, a syntax tree contains expressions, and a directory contains files and directories. A tree captures this hierarchy and gives us useful operations: visit every item in a chosen order, process a whole subtree, or follow a path from a root to one item.

The word “tree” alone does not promise fast lookup. A tree may be only a representation of nested data, with a search that visits every node. Faster operations require additional structure, such as a search-order invariant or a balancing rule. This chapter focuses on the shared model and traversal techniques. Chapter 83 covers heaps, Chapter 85 covers graphs more broadly, and Chapter 91 compares data structures against workload requirements.

## Mental Model

A rooted tree starts at one node and branches into child nodes. Every node other than the root has exactly one parent, and following child links never returns to an earlier node:

```text
Catalog                     root, depth 0
├── Books                   child of Catalog, depth 1
│   ├── Fiction             child of Books, depth 2
│   └── History             child of Books, depth 2
└── Games                   child of Catalog, depth 1
```

Each node is the root of its own subtree: that node plus all its descendants. This recursive shape is what makes trees convenient to process recursively, but also makes nesting depth and cycles important engineering concerns.

## Core Concept

Tree terminology is easiest to learn against one consistent convention:

- The **root** is the starting node and has no parent.
- A **parent** has one or more outgoing child relationships. A **child** is directly below its parent.
- A **leaf** has no children. Other nodes are internal nodes.
- A node's **depth** is the number of edges from the root to that node. The root has depth zero.
- A node's **height** is the greatest number of edges from it down to any leaf. A leaf has height zero.
- A **subtree** is a node together with all its descendants.
- The tree's **height** is the height of its root.

Some references count nodes instead of edges when defining height, and conventions for an empty tree vary. State the convention when values matter. In this chapter, height and depth count edges.

A strict rooted tree has one root, no cycles, and one parent per non-root node. Its links connect all nodes to the root. If one child is shared by two parents, or a child links back to an ancestor, the structure is a directed acyclic graph or a general graph rather than a strict tree. Those shapes need different rules; Chapter 85 develops the graph model.

### Shape and balance

Trees can have many shapes. A broad tree has many siblings; a deep tree has a long path. An unbalanced binary search tree can degenerate into a chain:

```text
10
  \
   20
     \
      30
        \
         40
```

The height of this four-node tree is three edges, so walking from the root to the last node takes work proportional to the number of nodes. A balanced tree keeps height small relative to its node count, which can make root-to-leaf operations logarithmic. “Balanced” does not name one universal invariant: AVL trees and red-black trees, for example, impose different rules. A shape that looks balanced in one small example does not guarantee a logarithmic bound; the implementation must maintain a defined rule.

Also separate shape from ordering. A plain hierarchy has no implied sort order. A binary search tree is a particular binary-tree variant that places values according to a comparison rule. Searching it costs O(h), where h is its height: O(log n) when the balancing invariant bounds height logarithmically, and O(n) in a degenerate tree. A tree that has no search-order invariant may require O(n) work to find a value.

## How It Works

### Traversal order

Traversal visits nodes systematically. The order affects output, side effects, and whether children are ready when a parent is processed.

For a binary tree, **pre-order** visits the node, then its left subtree, then its right subtree. **In-order** visits left subtree, node, then right subtree. **Post-order** visits the children before the node. In-order has a meaningful left/right definition only for a binary tree; there is no single in-order convention for arbitrary n-ary trees. **Level order** visits nodes breadth-first, completing one depth before moving to the next.

For this tree:

```text
        A
       / \
      B   C
     /
    D
```

pre-order is `A, B, D, C`; in-order is `D, B, A, C`; post-order is `D, B, C, A`; and level order is `A, B, C, D`.

Pre-order is useful when emitting a hierarchy in display order or copying parent data before descendants. Post-order is useful when a parent depends on results from its children, such as calculating subtree totals or removing nested resources from the leaves upward. Level order is useful when work should proceed by distance from a root. For a multi-child tree, define child order as part of the contract; the traversal follows that order.

Depth-first traversals naturally use the call stack or an explicit stack. Chapters 77 and 78 introduced those structures. Level-order traversal uses a FIFO queue, as in Chapter 78's breadth-first search. The distinction is the traversal policy: depth-first follows a branch before siblings, while level order visits all current siblings before their children.

### Recursive and iterative depth-first traversal

A small recursive traversal mirrors the recursive tree definition:

```php
<?php

declare(strict_types=1);

final class CategoryNode
{
    /** @var list<CategoryNode> */
    public array $children = [];

    public function __construct(
        public readonly string $id,
        public readonly string $label,
    ) {
    }

    public function addChild(self $child): void
    {
        $this->children[] = $child;
    }
}

/** @param list<string> $labels */
function appendPreorderLabels(CategoryNode $node, array &$labels): void
{
    $labels[] = $node->label;

    foreach ($node->children as $child) {
        appendPreorderLabels($child, $labels);
    }
}

/** @return list<string> */
function preorderLabels(CategoryNode $root): array
{
    $labels = [];
    appendPreorderLabels($root, $labels);

    return $labels;
}
```

The helper appends into one result list, avoiding a new result array at each recursive level. An explicit stack can instead make traversal state visible and avoid recursive calls:

```php
/** @return list<string> */
function preorderLabelsIteratively(CategoryNode $root): array
{
    $labels = [];
    $stack = [$root];

    while ($stack !== []) {
        /** @var CategoryNode $node */
        $node = array_pop($stack);
        $labels[] = $node->label;

        // Push in reverse so the first child is visited first.
        for ($index = count($node->children) - 1; $index >= 0; --$index) {
            $stack[] = $node->children[$index];
        }
    }

    return $labels;
}
```

The method assumes its input is a strict tree. Reversing the push order is necessary because a stack removes its most recently added item first. An iterative form avoids using one PHP function call per level, but it still needs memory for pending nodes; it does not make an extremely deep tree free.

### Level-order traversal

To produce a breadth-first listing, enqueue the root, then enqueue each node's children after removing that node from the front:

```php
/** @return list<string> */
function levelOrderLabels(CategoryNode $root): array
{
    $queue = new SplQueue();
    $queue->enqueue($root);
    $labels = [];

    while (!$queue->isEmpty()) {
        /** @var CategoryNode $node */
        $node = $queue->dequeue();
        $labels[] = $node->label;

        foreach ($node->children as $child) {
            $queue->enqueue($child);
        }
    }

    return $labels;
}
```

On a strict tree, each node is enqueued once. If relationships may contain cycles or shared child objects, add identity-based visited tracking before enqueueing, or validate and reject the malformed structure first. Otherwise the same object can be processed repeatedly or a cycle can keep the queue from becoming empty.

## PHP Representations

PHP does not provide a built-in general-purpose tree type. Choose a representation that makes the domain's invariants understandable and enforceable.

### Nested arrays

PHP arrays can contain arrays, so a small static hierarchy can be represented directly:

```php
$menu = [
    'label' => 'Shop',
    'children' => [
        ['label' => 'Books', 'children' => []],
        ['label' => 'Games', 'children' => []],
    ],
];
```

This is convenient for configuration, decoded data, and simple transformations. PHP arrays are ordered maps, not a specialized tree-node allocation; their semantics and memory trade-offs are discussed in Chapters 49, 50, 74, and 75. A nested array also does not automatically guarantee that required keys exist, that every `children` value is a list, or that arbitrary reference-based input is acyclic.

### Node objects

Objects are useful when a node has behavior, identity, or domain rules. The `CategoryNode` example makes child relationships explicit and lets application code name operations such as adding a child or calculating a path. The example exposes a mutable child list for clarity; production code should usually control mutation through an aggregate or builder that can enforce unique IDs, one parent per node, and no ancestor cycles.

Keep identity separate from domain equality. Two category objects with the same fields may compare equal with `==` but are not the same instance under `===`. PHP documents this distinction in its [object comparison rules](https://www.php.net/manual/en/language.oop5.object-comparison.php). For cycle detection or shared-reference detection, instance identity is often what matters; for persisted categories, the stable category ID is usually the domain key.

### Flat records and child indexes

Persistent data often arrives as rows such as `id`, `parent_id`, and `position`. Keeping rows flat can make updates and database queries straightforward. To traverse an already loaded snapshot, group child IDs or rows by parent ID, keeping root rows separately. Then start at a root and follow each parent's child list.

The index takes O(n) additional space for n rows and must be rebuilt or updated when the source changes. Validate that every non-root parent exists, root count matches the domain, each node has no more than one parent, and no parent chain loops. Make sibling order explicit with a stored position or sort key. Avoid using a nullable `parent_id` directly as an array key; represent roots separately or normalize keys deliberately. PHP array key conversion is covered in Chapter 75.

For a very large persistent hierarchy, do not assume that loading every row into PHP is the right design. Let the database own persistence and use a query strategy suited to the access pattern. Chapter 85 covers general graph representations; later database chapters discuss indexes and query plans.

## What PHP Does

The language supplies arrays and objects from which an application can build trees, plus structures such as `SplStack`, `SplQueue`, and `SplObjectStorage`. The PHP Manual describes `SplQueue` as a FIFO queue based on a doubly linked list and `SplObjectStorage` as a map from objects to data that can also be used as an object set. These classes provide behavior; the application still defines the tree shape and its invariants.

Object variables refer to object instances, which is why one object can be reachable through several child properties. `SplObjectStorage` can track visited instances without confusing two distinct objects that happen to have equal fields. Its visited set adds memory proportional to the number of recorded objects. If nodes are represented by scalar IDs or arrays instead, a visited map should use the domain's canonical ID and apply the same normalization rules throughout.

Avoid assuming a PHP array's concise syntax gives it the same memory profile as a packed tree node in a lower-level language. Each nested map and node value contributes allocation and metadata. See Chapter 74 for peak live memory and Chapter 56 for PHP-managed allocation. Measure realistic tree sizes and depths when they influence worker limits.

## Complexity and Memory

Let `n` be the number of reachable nodes, `h` the height in edges, and `w` the maximum number of nodes at any one depth.

- A full traversal is O(n) time when each node and child link is processed once. For a strict tree, there are n - 1 child links.
- A recursive depth-first traversal uses O(h) call state. A chain has h = n - 1, so recursive state can grow linearly.
- A stack-based preorder traversal that pushes every child at once can have O(n) pending-node space for a very wide tree. A frame-based traversal can keep O(h) frames, but should only be used when that smaller bound matters.
- A level-order traversal uses O(w) queue space in addition to its output. The widest level controls the queue peak.
- Returning all visited nodes or labels requires O(n) output space regardless of traversal method.
- A binary search tree operation follows one root-to-node path and costs O(h); the tree's invariant and balancing policy determine h.

These are structural bounds. PHP object/array overhead, result strings, allocator behavior, and any database hydration affect actual bytes and latency. Separate the traversal's working state from its returned output, as Chapter 74 recommends.

## Cycles, Shared Nodes, and Graph Boundaries

In a tree of object references, it is possible to accidentally attach the same child under two parents or attach an ancestor below its descendant. Either breaks the strict-tree invariant. A traversal written under the strict-tree assumption may then duplicate work, loop forever, or compute incorrect parent-dependent values.

If arbitrary object links must be inspected, track visited object identity. For example:

```php
$seen = new SplObjectStorage();

if (isset($seen[$node])) {
    throw new UnexpectedValueException(
        'The hierarchy contains a cycle or a shared node.',
    );
}

$seen[$node] = true;
```

In a validator, perform this check as each node is reached and also ensure the stored ID is unique. The example detects repeated instances, which covers a cycle or shared object; it does not detect two separate objects with the same business ID unless the validator also keeps an ID set. PHP's [SplObjectStorage documentation](https://www.php.net/manual/en/class.splobjectstorage.php) describes its object-set use. Chapter 76 introduced object-identity sets, and Chapter 85 explains how the visited-set rule extends when shared nodes are valid graph structure rather than invalid tree input.

Parent references deserve care. A child-to-parent pointer can be convenient, but together with parent-to-child references it creates a cycle in the PHP object-reference graph. PHP has cycle collection for unreachable circular references, as described in the [garbage collection manual](https://www.php.net/manual/en/features.gc.collecting-cycles.php), but collection does not release objects that remain reachable from a request, cache, or long-running worker. A parent ID or an external index may be a simpler ownership model when upward navigation is infrequent.

## Production Example: Rendering a Category Menu

Suppose a catalog menu must show categories in their configured sibling order. The tree model fits because each category has one parent and the menu renders a root and its descendants. A service can load a bounded category snapshot, build nodes in a first pass, and connect parent-child relationships in a second pass. That avoids requiring parents to appear before children in the query result.

Before publishing the menu, the builder should reject duplicate IDs, missing parent IDs, multiple roots when the domain requires one, and repeated nodes or cycles. It should preserve an explicit sibling position and escape labels at the HTML output boundary. Rendering should not depend on incidental database row order or on a process-local map surviving a worker restart.

For a small hierarchy, recursively rendering children may be clear and safe. For imported data with a user-controlled maximum depth, validate node count and depth before recursive rendering, or use an iterative traversal. If the menu is cached, tie cache invalidation to category changes; a fast traversal of stale categories still produces a wrong menu.

## Testing

Test both the shape contract and the operations built on it:

1. Cover a single root that is also a leaf, a chain, a wide root, and a tree with several levels.
2. Assert pre-order, post-order, and level-order results on a small tree whose expected sequence is obvious. Only assert in-order for a binary tree with a defined left/right meaning.
3. Verify sibling order, duplicate-ID policy, missing-parent handling, and root-count rules.
4. Try a repeated object reference and an ancestor cycle; validation should reject them before they reach a traversal that assumes a strict tree.
5. Generate small random trees and check useful properties: every node appears exactly once, pre-order starts with the root, post-order ends with the root, and level-order depths never decrease.
6. Test deep and wide inputs near the application's configured limits. Use a benchmark or memory test for capacity questions rather than fragile timing assertions in ordinary unit tests.

If a traversal invokes callbacks, test what happens when a callback throws and whether traversal can be resumed. Avoid performing irreversible side effects before a later validation step can fail; validate first, then publish or mutate external state.

## Common Mistakes

- Calling every nested structure a tree without checking the one-parent and no-cycle invariants.
- Assuming every tree has sorted values or fast lookup.
- Using “balanced” without naming the balancing rule or the operation it bounds.
- Applying in-order traversal to a general multi-child tree.
- Forgetting to push children in reverse when a stack should preserve left-to-right visitation.
- Traversing arbitrary object references without cycle or shared-node handling.
- Recursing through attacker-controlled depth without a depth or node-count limit.
- Returning a new array at every recursive step and ignoring intermediate memory.
- Treating object equality (`==`), object identity (`===`), and a domain ID as interchangeable.
- Building a child index without deciding how it is refreshed when records change.

## Senior Engineer Thinking

Start with the relationship contract: Is there one root or many? Can a node have multiple parents? Are child order and IDs meaningful? Can the structure change during traversal? These decisions determine whether a tree is the right model and what must be validated at the boundary.

Then match representation and algorithm to the workload. A nested array may be ideal for a tiny immutable menu; domain nodes may make invariants clearer in a mutable catalog; flat rows may fit persistence and batch imports. A traversal's O(n) time is usually unavoidable if every node must be rendered, while the choice of recursive calls, explicit stack, queue, and cached index affects memory and operational limits.

Keep the abstraction honest. If references can be shared, model a graph and define visited behavior. If lookup is central, establish and test a search invariant rather than expecting hierarchy alone to make search fast. If the data belongs in a database, do not duplicate it into every PHP worker without accounting for freshness and memory.

## Exercises

1. For the `Catalog` diagram in the mental model, write the pre-order and level-order sequences. Define a post-order sequence and explain why it differs.
2. Implement a post-order function that computes the number of descendant products under each category. State its time and auxiliary-space complexity.
3. Model a file hierarchy with nested arrays, node objects, and flat `id`/`parent_id` records. Compare validation, update, traversal, and memory trade-offs.
4. Build a validator that detects duplicate category IDs, missing parents, a shared child object, and a cycle. Keep object identity and domain-ID checks separate.
5. Given a binary search tree with height `h`, explain the lookup cost. What additional invariant would be needed to claim logarithmic height?

## Review Questions

1. How do depth and height differ, and what convention does this chapter use?
2. What makes a rooted structure a strict tree? What changes when child nodes are shared?
3. Which traversal uses a queue, and which use a stack or call state?
4. Why is in-order traversal specific to binary trees?
5. Why does an ordinary tree traversal not automatically provide fast lookup?
6. How do height and maximum width affect depth-first and level-order memory?
7. When should cycle detection use object identity, and when should it use a domain ID?
8. What can a tree representation say about data ownership and persistence, and what can it not say?

## Summary

A rooted tree models one-parent, acyclic hierarchy. Root, parent, child, leaf, depth, height, and subtree describe its shape; balance and search order are separate invariants that must be defined by the chosen tree variant. Pre-order, post-order, in-order for binary trees, and level order serve different tasks. PHP arrays, node objects, and flat records can all represent trees, with different validation, identity, memory, and update costs. Traversal is O(n), but working memory depends on height, width, output, and any visited set. If PHP object references can form cycles or share nodes, validate that structure as a graph or reject it before treating it as a tree.

## References

- [PHP Manual: Arrays](https://www.php.net/manual/en/language.types.array.php)
- [PHP Manual: Comparing Objects](https://www.php.net/manual/en/language.oop5.object-comparison.php)
- [PHP Manual: `SplObjectStorage`](https://www.php.net/manual/en/class.splobjectstorage.php)
- [PHP Manual: `SplQueue`](https://www.php.net/manual/en/class.splqueue.php)
- [PHP Manual: Collecting Cycles](https://www.php.net/manual/en/features.gc.collecting-cycles.php)
- [Chapter 43 — AST](../04-php-under-the-hood/043-ast.md)
- [Chapter 74 — Memory Complexity](./074-memory-complexity.md)
- [Chapter 77 — Stacks](./077-stacks.md)
- [Chapter 78 — Queues](./078-queues.md)
- [Chapter 83 — Heaps](./083-heaps.md)
- [Chapter 85 — Graphs](./085-graphs.md)
