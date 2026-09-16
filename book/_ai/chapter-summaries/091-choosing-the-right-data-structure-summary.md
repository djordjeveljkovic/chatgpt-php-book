# AI Summary — Chapter 91 — Choosing the Right Data Structure

- Status: complete
- Volume: Volume 6 — ALGORITHMS AND DATA STRUCTURES
- Last updated: 2026-09-15

## Written material

Complete chapter presents a workload-based decision method using required operations and their frequency, identity and order contracts, collection growth and updates, peak memory, lifetime, and durability. A comparison table maps operation shapes to initial candidates and trade-offs. The PHP example builds an ID map, rejects duplicate IDs, orders values deterministically with `uasort()`, and retains key lookup. Database/process ownership, testing, operations, common mistakes, exercises, review questions, and summary are included.

## Concepts already explained

Build/query/update cost model, operation frequency, data-structure invariant, scalar versus object identity, duplicate policy, map/order dual use, composite representations, peak live memory, index freshness, source ownership, and process-local versus durable state.

## Terminology established

The preferred structure is the simplest candidate that satisfies the full workload contract. Cost comparison includes construction, repeated query cost, updates, retained and temporary memory, output, and external calls. A local index or queue does not imply freshness, cross-process coordination, or durability.

## Examples used

Comparison table for lists, maps/sets, stacks/queues, scans, sorted lists, heaps, trees/graphs, interval/window structures, streams/batches, memo tables, and persistent services. `indexAndOrderEvents()` combines exact string-ID map access with deterministic chronological iteration.

## Cross-references

Draws on Chapters 72–90: algorithm and Big O reasoning, memory, arrays/maps/sets, stacks/queues, sorting/searching, trees/heaps/priority queues/graphs, intervals/windows, batching, memoization, and the distinction between streaming and materialization. Also links to Chapter 244 for durable queue guarantees and to official PHP Manual pages for arrays, `array_key_exists()`, and `uasort()`.

## Open threads

No open chapter work remains.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

The PHP example linted on PHP 8.5.10. Runtime checks passed for chronological order, deterministic equal-time ID tie-breaking, lookup after `uasort()`, numeric-looking string ID preservation, empty input, duplicate rejection, and malformed-record rejection. Independent proofreading found no blocking defects. All local links resolve; `git diff --check` passes. Complexity entries are explicitly typical abstract models, not PHP API guarantees.

## Writing notes

Use the comparison table as a candidate map, not a fixed prescription. Include build/update costs and ownership boundaries. PHP arrays are ordered maps, so normalization and key encoding remain part of the domain contract; `uasort()` keeps keys with their values.
