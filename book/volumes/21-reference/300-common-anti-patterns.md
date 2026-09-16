---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 300
title: Common Anti-Patterns
slug: common-anti-patterns
status: complete
summary: ../../_ai/chapter-summaries/300-common-anti-patterns-summary.md
---

# Chapter 300 — Common Anti-Patterns

## Why This Matters

An anti-pattern is a recurring solution shape whose hidden costs exceed its benefit in the context where it is used. It is not a synonym for “code I dislike.” The same technique can be reasonable at one scale and harmful at another.

## How to Evaluate One

For each shape, identify its attraction, violated boundary, measurable cost, safer migration seam, and conditions under which the correction would be worse. Ask which invariant, quality attribute, or ownership rule is being weakened.

## Architecture and Object Design

### Distributed systems by fashion

Splitting a modular monolith into services can add network failure, deployment coordination, data duplication, and operational ownership without solving a real bottleneck. First establish independent ownership, scaling, security, or failure-isolation pressure. A module with explicit contracts may be the safer boundary.

### Interface proliferation

Adding an interface for every class creates indirection without a substitution need. Introduce a port where a boundary has a real alternate implementation, effect, test seam, or ownership change. Otherwise a concrete class can be the clearer contract.

### The “service” class that owns everything

A large class named `*Service` can conceal mixed responsibilities, transaction scope, authorization, formatting, and external effects. Split by capability and effect boundary, not by renaming methods. Preserve a narrow application operation while moving policy and adapters to explicit owners.

### Anemic domain objects everywhere

Passive records are useful for transport and persistence. They become harmful when every rule lives in controllers or scripts and invariants have no owner. Put durable decisions near the domain or database boundary while keeping persistence models honest about their role.

## Data and Framework Usage

### ORM pass-through as architecture

A repository that merely forwards every ORM method is not automatically an abstraction. It can hide generated SQL and make transaction ownership unclear. Define a purpose-specific query or domain port when it protects a contract; otherwise expose the query deliberately and inspect its plan.

### Cache as source of truth

A cache is often treated as authoritative because it is fast. This creates data-loss and invalidation risk. Name the authority, define stale behavior, scope keys by tenant and policy, and make rebuilds possible.

### Global mutable state

Globals, static registries, ambient sessions, and process-wide configuration make request order and worker lifetime part of correctness. Pass dependencies explicitly, reset long-lived state, and make lifecycle ownership visible.

### Framework magic without a contract

Convenient lifecycle hooks, auto-discovery, and implicit model behavior can be productive until a migration, worker, or CLI path behaves differently. Document the hook’s inputs, side effects, runtime assumptions, and tests. Framework convention is not a substitute for an application boundary.

## Reliability and Performance

### Retry amplification

Independent retries at HTTP, queue, database, and provider layers can multiply traffic during an outage. Choose one owner for the retry budget, classify errors, use idempotency, add jitter, and stop when recovery evidence is unavailable.

### Unbounded fan-out

Launching a request or job for every row can exhaust memory, connections, provider quotas, and worker time. Bound concurrency, batch work, record progress, and define partial-failure recovery.

### Synchronous everything

Making email, exports, indexing, and provider calls synchronous can make user latency equal to the slowest dependency. Asynchrony is not free: use it when the user contract permits delayed completion and provide durable status and reconciliation.

### Premature caching and micro-optimization

Optimizing before measuring can add invalidation and memory costs while the actual bottleneck is SQL, locking, or a provider. Establish a workload, target, baseline, and abort condition first.

## Testing and Delivery

### Mocking the system into success

Mocks can validate an interaction while hiding serialization, SQL, locking, timeout, or provider behavior. Keep unit tests focused, then test important real boundaries with controlled fixtures and failure injection.

### Big-bang rewrite

A rewrite discards operational knowledge and expands the unknown surface. Prefer characterization, seams, incremental replacement, dual reads, or a bounded capability migration unless evidence justifies the irreversible risk.

### Branches that never die

Permanent feature flags and compatibility adapters create multiple behaviors nobody tests completely. Record an owner, removal condition, telemetry, and deadline; delete the path once evidence permits it.

## A Decision Table

| Shape | Attraction | Hidden cost | Safer question |
| --- | --- | --- | --- |
| service split by folder | looks modular | no ownership or failure boundary | what capability needs isolation? |
| cache everything | lower read latency | stale scope and invalidation | what is authoritative and rebuildable? |
| retry every error | apparent resilience | amplification and duplicates | which failures are safe to retry? |
| mock every dependency | fast tests | false integration confidence | which real contract needs testing? |
| rewrite the legacy app | clean design | lost behavior and long exposure | what seam can reduce risk now? |
| add workers | more throughput | shared-resource saturation | what bottleneck and headroom exist? |

## Exercises

1. Identify three anti-patterns in a proposed service decomposition and name the evidence required.
2. Turn a cache-as-authority design into an explicit source-of-truth and rebuild plan.
3. Find retry layers in a request-to-provider workflow and assign one budget owner.
4. Design a seam for replacing one legacy capability without a big-bang rewrite.
5. Choose an anti-pattern whose correction would be inappropriate at small scale and explain why.

## Review Questions

- What distinguishes an anti-pattern from a style preference?
- When is an interface useful rather than ceremonial?
- Why is a cache not automatically an authority?
- How can retries amplify an outage?
- What makes a migration seam safer than a rewrite?
- Which evidence would cause you to retain the current design?

## Summary

Common anti-patterns are context failures: service splitting without ownership, interfaces without substitution, ORM pass-through, caches as authorities, global state, framework magic without contracts, retry amplification, unbounded fan-out, synchronous external work, mock-only testing, rewrites, and permanent flags. Correct them with evidence, explicit boundaries, bounded work, and reversible migration.

## Chapter 301 Handoff

After recognizing harmful solution shapes, Chapter 301 provides a data-structure selection guide based on operation, ordering, memory, ownership, and scale.

## References

- [Chapter 192 — Simple Architecture](../13-architecture/192-simple-architecture.md)
- [Chapter 122 — Query Builders](../08-databases/122-query-builders.md)
- [Chapter 223 — Performance Mental Model](../15-performance/223-performance-mental-model.md)
- [Chapter 290 — Technical Debt](../20-senior-engineering/290-technical-debt.md)
- [Chapter 296 — Engineering Trade-Offs](../20-senior-engineering/296-engineering-trade-offs.md)
