---
book: The Complete Modern PHP Engineering Book
volume: 6
volume_title: ALGORITHMS AND DATA STRUCTURES
chapter: 77
title: Stacks
slug: stacks
status: complete
summary: ../../_ai/chapter-summaries/077-stacks-summary.md
---

# Chapter 77 — Stacks

## Why This Matters

Some problems have a natural “most recent unfinished thing first” rule. A parser must close the most recently opened group. An undo feature must reverse the newest action before older ones. A depth-first traversal can postpone branches and return to the newest one. These are stack-shaped problems.

A stack makes that order explicit. It prevents code from searching for an arbitrary item or removing an item from the middle when the requirement says only the newest item is eligible. This constraint often simplifies both the algorithm and its invariants.

The term *stack* can also mean the call stack maintained by the runtime. That is related to the same last-in, first-out idea, but this chapter focuses on a userland data structure. A PHP array or an `SplStack` does not expose or replace the Zend Engine’s function-call frames. See [Chapter 46 — Zend VM](../04-php-under-the-hood/046-zend-vm.md) and [Chapter 54 — Function Calls](../04-php-under-the-hood/054-function-calls.md) for the runtime call path.

## Mental Model

Think of a pile where new items go on top and only the top item can be removed or inspected:

```text
             top
              ↓
          ┌───────┐
          │ task C│  ← newest
          ├───────┤
          │ task B│
          ├───────┤
          │ task A│  ← oldest
          └───────┘
```

The essential operations are:

```text
push(x)   add x to the top
pop()     remove and return the top item
peek()    inspect the top item without removing it
isEmpty() report whether there is a top item
```

If `A`, then `B`, then `C` are pushed, pops return `C`, `B`, then `A`. This is last in, first out (LIFO). The operations describe an abstract stack contract; they do not dictate whether the implementation uses an array, linked nodes, or another representation.

## Core Concept

Like the set in [Chapter 76](076-sets.md), a stack is an abstract contract rather than a mandated PHP representation. The difference is the question it answers: a set answers membership, while a stack answers which item comes next under a LIFO rule. Its invariant is that the next item removed is the most recently pushed item that has not already been removed. A stack deliberately does not promise efficient arbitrary lookup or removal. If an algorithm needs to inspect everything, iteration costs time proportional to the number of items; if it needs to remove an older item directly, a stack may be the wrong abstraction.

For a list-backed implementation, the right end can serve as the top:

```php
$stack = [];

$stack[] = 'parse expression'; // push
$stack[] = 'parse nested group';

$next = array_pop($stack);      // 'parse nested group'
```

An empty stack is a boundary condition, not a value to guess at. `array_pop()` returns `null` when the array is empty, but `null` may itself be a valid payload. Check emptiness before popping when that distinction matters.

## How It Works

### Matching nested delimiters

Balanced delimiters illustrate why LIFO order is useful. When an opening delimiter appears, remember it. When a closing delimiter appears, it must match the most recently opened delimiter that has not yet been closed. A counter alone cannot check mixed delimiter types such as `([)]`; a stack can.

```php
<?php
declare(strict_types=1);

function hasBalancedDelimiters(string $input): bool
{
    $openingFor = [
        ')' => '(',
        ']' => '[',
        '}' => '{',
    ];

    $open = [];
    $length = strlen($input);

    // This function expects delimiter text or already-tokenized input.
    // It treats all other bytes as ordinary text.
    for ($i = 0; $i < $length; $i++) {
        $character = $input[$i];

        if ($character === '(' || $character === '[' || $character === '{') {
            $open[] = $character;
            continue;
        }

        if (!isset($openingFor[$character])) {
            continue;
        }

        if ($open === []) {
            return false; // closing delimiter with nothing to match
        }

        $mostRecentOpening = array_pop($open);
        if ($mostRecentOpening !== $openingFor[$character]) {
            return false; // wrong delimiter type or nesting order
        }
    }

    return $open === []; // leftover openings were never closed
}
```

The stack records the unmatched opening delimiters, not every character. For `([{}])`, each closing token matches and removes the most recent opening. For `([)]`, `)` attempts to close `[` and fails immediately. An extra closer fails when the stack is empty; an extra opener remains at the end.

The loop processes `n` input bytes once, so time is O(n). The stack holds at most `d` unmatched delimiters, where `d` is maximum nesting depth, so auxiliary space is O(d). In the worst case, `d` grows with `n`.

This is a delimiter checker, not a PHP parser. It will count brackets inside a quoted string or comment unless those tokens were removed first. Real source parsing needs lexical rules for strings, comments, heredocs, and other syntax. PHP’s lexer/parser already handles PHP source; use a proper parser or tokenizer when analyzing it instead of applying this small routine to raw source text.

### Undo and history

An undo history also follows LIFO order: the latest reversible action is undone first. A common design stores an inverse operation or a snapshot when applying each new action. Undo pops one history entry and applies its inverse; redo usually uses a second stack. A new action after an undo generally clears the redo stack because it creates a new history branch.

That model has practical limits. A stack of full document snapshots can consume O(k × s) memory for `k` snapshots of size `s`. Storing compact commands can save space, but commands must retain enough data to reverse their effects. Cap retained history, store deltas where appropriate, and make the meaning of each command stable.

An application-level undo is not automatically a rollback of external effects. Sending an email, charging a card, or publishing a message cannot usually be undone by restoring an in-memory snapshot. Those actions need an explicit compensating action or a domain workflow. For durable history, persist ordered events or versions and use database transactions and concurrency controls; an in-process stack disappears with its PHP process and cannot coordinate simultaneous requests.

## What PHP Does

### A PHP array can act as a stack

PHP arrays are ordered maps, as discussed in [Chapter 75 — Arrays and Hash Maps](075-arrays-and-hash-maps.md). For a simple stack, keep them as a list: append with `$stack[] = $value` and remove from the end with `array_pop($stack)`. Avoid arbitrary string keys and front insertion; they do not help the stack contract and can change the behavior and cost of operations.

This representation is concise and familiar. It is often a good default when the stack is local to one algorithm. Do not treat the complexity model as a performance guarantee in the language contract: the PHP Manual documents what `array_pop()` returns and changes, not a formal Big O bound. For a typical packed-list use, end operations are the intended shape; benchmark if the stack is large or performance-sensitive.

Encapsulate the array when empty behavior or invariants deserve a named API:

```php
<?php
declare(strict_types=1);

final class Stack
{
    /** @var list<mixed> */
    private array $items = [];

    public function isEmpty(): bool
    {
        return $this->items === [];
    }

    public function push(mixed $value): void
    {
        $this->items[] = $value;
    }

    public function peek(): mixed
    {
        if ($this->isEmpty()) {
            throw new UnderflowException('Cannot inspect an empty stack.');
        }

        return $this->items[array_key_last($this->items)];
    }

    public function pop(): mixed
    {
        if ($this->isEmpty()) {
            throw new UnderflowException('Cannot remove an item from an empty stack.');
        }

        return array_pop($this->items);
    }
}
```

The class chooses an explicit underflow error, while the raw `array_pop()` API returns `null` on empty input. Because `mixed` permits `null`, callers should use `isEmpty()` before `peek()` or `pop()` when a null payload is possible. A richer result type can make “empty” and “popped null” distinct in a single return value, but it is unnecessary when a checked operation is clear.

### `SplStack`

The Standard PHP Library provides `SplStack`, whose documented implementation is based on `SplDoublyLinkedList` configured for LIFO iteration. It exposes stack-oriented methods such as `push()`, `pop()`, `top()`, and `isEmpty()`:

```php
$stack = new SplStack();
$stack->push('first');
$stack->push('second');

if (!$stack->isEmpty()) {
    $latest = $stack->top(); // 'second'; does not remove it
    $removed = $stack->pop(); // 'second'
}
```

`SplStack` can make the intent visible in code and provides an object with stack operations. It also inherits behavior from `SplDoublyLinkedList`, including configurable iteration modes and indexed access. That broader surface is useful in some cases and more API than others need. Its linked-node representation has different allocation and memory costs from an array. Neither representation is universally faster; choose based on the contract you need and measure the actual workload.

Iteration over an `SplStack` follows LIFO order, so it visits the top item first. The inherited iterator mode can also be configured to delete visited items. If iteration is part of correctness, state whether it is observational or consuming and test that behavior explicitly. Do not assume an iterator is equivalent to repeatedly calling `pop()` unless the selected mode and intended side effects agree.

### A userland stack is not the runtime call stack

Nested PHP function calls use runtime-managed execution state. Recursion creates more call depth; a userland stack stores values that application code pushes itself. Replacing recursion with an explicit stack can make traversal depth visible and let code control pending work, but it does not eliminate the memory cost of retaining that work. Conversely, creating an `SplStack` does not alter Zend VM call frames.

## Performance and Memory

For the abstract stack, `push`, `pop`, and `peek` are expected O(1) operations; visiting all `n` elements is O(n). With a list-backed PHP array, appending and popping at the end usually fit that model for this use. A linked-node stack also supports constant-time end operations under the standard linked-list model. PHP’s manual specifies API behavior rather than a portable complexity guarantee, so treat Big O as the data-structure model and confirm real costs with measurements.

The space bound is O(h), where `h` is the maximum number of live entries, plus the memory for the values those entries keep reachable. [Chapter 74 — Memory Complexity](074-memory-complexity.md) distinguishes auxiliary space from input and peak live memory. In PHP, each array entry has hash-table overhead; a node-based structure also needs node/link metadata and allocations. If entries are objects, strings, or large snapshots, the payload may dominate either structure.

An array assignment may initially share storage through copy-on-write, as covered in [Chapter 52 — Copy-on-Write](../04-php-under-the-hood/052-copy-on-write.md). Mutating a shared array can require separation, so copying a large stack for a snapshot can be cheap at first and more expensive when either copy changes. Long-running workers should also release completed work and bound retained history; values still referenced by a stack remain live. See [Chapter 65 — Long-Running PHP Processes](../05-php-runtime/065-long-running-php-processes.md) for worker lifetime concerns.

## Testing

Test the stack’s contract rather than its storage details:

- Pushing `a`, `b`, and `c` then popping returns `c`, `b`, and `a`.
- `peek()` returns the top without changing the next value that `pop()` returns.
- The empty-stack policy is consistent for both `peek()` and `pop()`.
- A `null` payload is not mistaken for an empty stack.
- The delimiter checker accepts empty and properly nested input, and rejects early closers, leftover openers, and mismatched nesting such as `([)]`.
- If traversal must preserve a stack, verify that iteration leaves its contents unchanged.

Property-based tests can generate push/pop sequences and compare the implementation against a simple reference list. After every operation, the stack top should equal the last reference-list item, if one exists. For the delimiter checker, generate valid nested strings and then mutate a delimiter to verify that malformed cases are rejected.

## Common Mistakes

- Using a counter to validate several kinds of nested delimiters, losing delimiter type and nesting order.
- Popping without defining empty-stack behavior.
- Treating `null` from `array_pop()` as unambiguous when `null` is also a valid item.
- Using a stack when older entries need frequent arbitrary lookup or removal.
- Assuming a stack of snapshots is cheap because each push is one line of code.
- Treating in-memory undo as reversal of database commits or external side effects.
- Applying delimiter checks to raw source code without accounting for strings and comments.
- Confusing the data structure with PHP’s runtime-managed call stack.

## Senior Engineer Thinking

Choose a stack when the domain rule itself is LIFO. Make the underflow policy explicit, bound the maximum retained work, and identify what each entry keeps alive. For parsing, be clear about the token stream the algorithm receives. For undo, separate reversible local state from durable or external effects. For an iterative traversal, define what happens when the graph is very large and whether nodes or pending branches remain reachable after completion.

The structure is a small part of the design. Correctness comes from matching the LIFO invariant to the problem and preserving it across errors, retries, request boundaries, and persistence where those boundaries apply.

## Exercises

1. Extend the delimiter checker to report the byte offset of the first invalid closing delimiter and the offset of the first unclosed opener at end of input.
2. Implement a depth-first traversal of nested arrays using an explicit stack. Define whether child order is preserved and test a deeply nested input.
3. Design undo and redo stacks for a text editor command. Explain why a new command after undo changes redo history, and set a memory limit for retained history.
4. Benchmark an array-backed stack and `SplStack` with realistic values and operation counts. Record PHP version, peak memory, and whether the benchmark includes construction and iteration.

## Review Questions

1. What is the defining invariant of a stack, and which operations does the abstract contract expose?
2. Why can a counter validate one delimiter type but not arbitrary mixed nesting?
3. What is the worst-case auxiliary space of delimiter matching, and what input dimension determines it?
4. How do a plain PHP array and `SplStack` differ in API and representation?
5. Why should complexity claims about PHP’s implementation be qualified even when the abstract stack operation is O(1)?
6. Why is an in-memory undo stack insufficient for reliably reversing a payment or coordinating concurrent requests?

## Summary

A stack is an abstract LIFO structure: push adds to the top, pop removes the newest remaining item, and peek observes that item. It fits nested parsing, undo histories, and depth-first work. In PHP, a list-shaped array is a lightweight implementation for many local algorithms; `SplStack` offers a dedicated stack API backed by a doubly linked list. Define empty behavior, account for retained memory, distinguish runtime call frames from userland data, and measure implementation costs when they matter.

## References

- [PHP Manual: `array_pop()`](https://www.php.net/manual/en/function.array-pop.php)
- [PHP Manual: The `SplStack` class](https://www.php.net/manual/en/class.splstack.php)
- [PHP Manual: The `SplDoublyLinkedList` class](https://www.php.net/manual/en/class.spldoublylinkedlist.php)
