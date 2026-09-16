---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 304
title: Architecture Decision Guide
slug: architecture-decision-guide
status: complete
summary: ../../_ai/chapter-summaries/304-architecture-decision-guide-summary.md
---

# Chapter 304 — Architecture Decision Guide

## Why This Matters

Architecture is a set of boundaries and decisions about ownership, data, effects, deployment, and failure. A folder structure or a class called `Service` does not create isolation. Choose the smallest architecture that protects the required qualities and can be operated by the team.

## Frame the Decision

Record the capability, actors, trust boundaries, source of truth, quality targets, workload, failure tolerance, ownership, migration window, and reversibility. Include inaction and containment as alternatives. A decision without a review condition becomes an undocumented permanent constraint.

## Boundary Types

| Boundary | What it can protect | What it costs |
| --- | --- | --- |
| function/class | local policy and testability | little isolation from shared state |
| module | dependency direction and ownership | discipline and internal contracts |
| process/service | deployment, scaling, and failure isolation | network, data, and operational complexity |
| queue workflow | latency and retry separation | eventual consistency and reconciliation |
| projection/cache | read capacity and query shape | freshness, rebuild, and invalidation |

Choose a deployable boundary only when its independent lifecycle is valuable enough to pay for network failure, versioning, observability, and ownership.

## Architecture Choices

### Simple or layered application

A simple application can keep request handling, policy, persistence, and effects close together when the capability is small and the transaction boundary is clear. Layers help when they make dependencies and tests clearer; ceremonial layers add navigation without protection.

### Modular monolith

Use modules when several capabilities need explicit ownership and contracts but do not yet need independent deployment. Enforce inward dependencies, prevent arbitrary table access, and give each module clear transaction and event boundaries. A modular monolith can be a destination, not merely a failed service decomposition.

### Hexagonal or ports-and-adapters boundary

Use a port when the application must remain independent of a database, provider, clock, queue, or framework lifecycle. Keep the port meaningful: it should protect a decision or effect contract, not mirror every vendor method.

### Queue and projection

Move work to a queue when the user contract permits delayed completion, the work can be identified and retried, and durable status/reconciliation exists. Add a projection when a read shape, freshness window, rebuild process, and source authority are explicit. Eventual consistency is a user-visible contract, not an implementation footnote.

### Service extraction

Extract a service for independent ownership, deployment, scaling, security boundary, or failure isolation. Define data ownership, API/message contracts, timeouts, idempotency, observability, version compatibility, and migration. “The module is large” is evidence for investigation, not a sufficient boundary criterion.

## Data and Transaction Ownership

Every durable decision should have one authoritative owner. Cross-boundary transactions should be replaced with explicit workflows, outbox/inbox records, idempotent effects, and reconciliation where appropriate. Do not call a cache or read model authoritative merely because clients read it most often.

State where authorization occurs, how tenant scope travels through caches and jobs, and which component may mutate each record. Shared tables and direct cross-module writes are architecture decisions that deserve the same scrutiny as APIs.

## Failure and Operations

For each boundary, list timeout, retry, partial deployment, stale data, duplicate message, provider uncertainty, dependency outage, and rebuild behavior. Name metrics, logs, traces, queue age, freshness, saturation, and ownership. An architecture that cannot be diagnosed or recovered is incomplete.

## Migration and Reversibility

Prefer seams that permit characterization, shadow reads, dual compatibility, one write authority, bounded backfill, canary rollout, and retirement evidence. A service split can be staged; a public contract, data duplication, or message schema may be difficult to undo. State rollback limits and forward-recovery actions before rollout.

## Decision Record

```text
Decision:
Capability and owner:
User-visible goal:
Invariants and trust boundaries:
Options, including inaction:
Chosen boundary and rationale:
Data/source-of-truth owner:
Failure, retry, and unknown-completion behavior:
Observability and capacity evidence:
Migration and compatibility plan:
Rollback or forward recovery:
Accepted risks and owner:
Review trigger/date:
```

## Common Mistakes

- choosing microservices by fashion;
- treating namespaces as isolation;
- ignoring data ownership and transaction scope;
- adding queues without status or reconciliation;
- treating projections as authorities;
- introducing ports that mirror frameworks;
- omitting timeout, retry, and observability design;
- planning extraction without a migration seam;
- declaring rollback possible without checking schemas and messages.

## Exercises

1. Compare a modular monolith and service split for a reservation capability.
2. Draw data, effect, and ownership boundaries for a search projection.
3. Turn a synchronous notification into an explicit queue workflow.
4. Write an ADR including inaction, review trigger, and forward recovery.
5. Identify which parts of a proposed architecture are reversible.

## Review Questions

- What makes a boundary real?
- When is a modular monolith the better architecture?
- What must be explicit before adding a queue or projection?
- Why is data ownership more important than naming?
- Which architecture decisions are hard to reverse?
- What operational evidence belongs in an ADR?

## Summary

Architecture decisions should begin with capability, ownership, invariants, trust boundaries, quality targets, workload, failure, and reversibility. Modules, ports, queues, projections, and services each protect different qualities and impose different costs. Choose proportionate boundaries, make data and recovery explicit, and migrate through observable seams.

## Chapter 305 Handoff

Architecture determines where data decisions live. Chapter 305 provides a database decision guide for choosing between PHP, SQL, indexes, transactions, locks, replicas, queues, and read models.

## References

- [Chapter 192 — Simple Architecture](../13-architecture/192-simple-architecture.md)
- [Chapter 194 — Modular Monolith](../13-architecture/194-modular-monolith.md)
- [Chapter 195 — Hexagonal Architecture](../13-architecture/195-hexagonal-architecture.md)
- [Chapter 206 — Distributed Systems](../13-architecture/206-distributed-systems.md)
- [Chapter 290 — Technical Debt](../20-senior-engineering/290-technical-debt.md)
- [Chapter 296 — Engineering Trade-Offs](../20-senior-engineering/296-engineering-trade-offs.md)
