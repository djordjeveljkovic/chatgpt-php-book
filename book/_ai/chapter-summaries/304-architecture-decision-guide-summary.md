# AI Summary — Chapter 304 — Architecture Decision Guide

- Status: complete
- Volume: Volume 21 — REFERENCE
- Last updated: 2026-09-17

## Written material

Chapter 304 guides architecture decisions from capability, ownership, invariants, trust boundaries, quality targets, workload, failure, operations, migration, and reversibility. It compares simple/layered apps, modular monoliths, ports, queues, projections, caches, service extraction, data ownership, a decision record, exercises, and the Chapter 305 handoff.

## Concepts already explained

Capability; boundary; ownership; source of truth; module versus service; port; eventual consistency; queue; projection; reversibility; forward recovery; review trigger.

## Terminology established

Reservation capability, search projection, notification workflow, service extraction, cache, queue, migration seam.

## Examples used

None.

## Cross-references

[Chapter 192 — Simple Architecture](../../volumes/13-architecture/192-simple-architecture.md); [Chapter 194 — Modular Monolith](../../volumes/13-architecture/194-modular-monolith.md); [Chapter 195 — Hexagonal Architecture](../../volumes/13-architecture/195-hexagonal-architecture.md); [Chapter 206 — Distributed Systems](../../volumes/13-architecture/206-distributed-systems.md); [Chapter 290 — Technical Debt](../../volumes/20-senior-engineering/290-technical-debt.md); [Chapter 296 — Engineering Trade-Offs](../../volumes/20-senior-engineering/296-engineering-trade-offs.md).

## Open threads

Continue with Chapter 305 — Database Decision Guide.

## Exact next section

Chapter 305 — Database Decision Guide: the Why This Matters section.

## Technical verification notes

The source contains an ADR template and prose with no executable PHP blocks. Local Markdown links resolved and `git diff --check` passed. The chapter received a local editorial check for boundary selection, ownership, source of truth, consistency, failure, observability, migration, and reversibility. Live architecture, deployment, queue, and database integrations were not run.

## Writing notes

Keep this summary short and update it after every writing session.
