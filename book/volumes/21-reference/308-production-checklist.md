---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 308
title: Production Checklist
slug: production-checklist
status: complete
summary: ../../_ai/chapter-summaries/308-production-checklist-summary.md
---

# Chapter 308 — Production Checklist

## Why This Matters

Production readiness is the ability to deliver, observe, operate, recover, and retire a capability while preserving user and data contracts. A green build or live process is not proof that the service is ready.

## Artifact and Configuration

- [ ] source, dependency lock file, generated autoloaders, migrations, and configuration are versioned or provenance-linked;
- [ ] artifact identity is recorded and promoted immutably;
- [ ] PHP binary, SAPI, extensions, ini settings, Composer platform, and OPcache policy match the runtime matrix;
- [ ] secrets are injected, least-privileged, rotatable, and absent from logs;
- [ ] old/new code, schema, cache values, and queue messages are compatible during rollout;
- [ ] feature flags have owners, defaults, telemetry, and removal criteria.

## Schema and Data

- [ ] migrations are reviewed for locks, duration, replication, and old-worker compatibility;
- [ ] expand-and-contract sequencing exists for incompatible changes;
- [ ] backfills are bounded, resumable, observable, and rate-limited;
- [ ] constraints, tenant scope, authorization, and audit behavior are verified;
- [ ] caches and projections have authority, freshness, rebuild, and reconciliation plans;
- [ ] backup and restore evidence covers the changed data.

## Health, Traffic, and Workers

- [ ] liveness, readiness, dependency health, and capability health are distinct;
- [ ] smoke tests cover authenticated, authorized, tenant-scoped, and degraded paths;
- [ ] FPM limits, database connections, queues, memory, timeouts, and provider quotas have headroom;
- [ ] CLI, cron, queue, migration, repair, and scheduled jobs are included in the rollout;
- [ ] graceful shutdown, drain, acknowledgement, retry, poison-message, and worker restart behavior are tested;
- [ ] traffic can be canaried, paused, shed, or routed to a safe mode.

## Observability

- [ ] metrics cover latency percentiles, errors, saturation, queue age, retries, freshness, and business invariants;
- [ ] structured logs include correlation and deployment identity while redacting sensitive data;
- [ ] traces or equivalent evidence connect FPM, SQL, cache, queue, and provider phases;
- [ ] alerts have thresholds, owners, runbooks, escalation, and maintenance behavior;
- [ ] dashboards compare the new cohort with a known baseline;
- [ ] operators can distinguish failed, not started, completed, and completion unknown.

## Rollout, Rollback, and Recovery

- [ ] canary scope, promotion gates, abort conditions, and decision owners are explicit;
- [ ] rollback considers schema, messages, caches, projections, credentials, and external effects;
- [ ] forward recovery and reconciliation exist when rollback is unsafe;
- [ ] recovery points and recovery time targets are understood and restore-tested;
- [ ] incident command, evidence preservation, communication, and customer-impact handling are ready;
- [ ] post-release verification checks user-visible behavior, data integrity, security, capacity, and cleanup.

Rollback is a capability, not a button. A previous binary may be unable to read a new schema or undo an external effect. Do not declare rollback ready without checking those boundaries.

## Retirement

- [ ] temporary flags, compatibility adapters, emergency access, dashboards, and runbooks have owners and removal dates;
- [ ] unused queues, indexes, projections, credentials, and data are retired safely;
- [ ] support and incident documentation reflect the final architecture;
- [ ] capacity and cost are remeasured after traffic converges.

## Release Record

```text
Artifact and version:
Runtime/SAPI/extensions:
Schema/message compatibility:
Canary and promotion gates:
Owner and on-call:
Key dashboards and alerts:
Rollback limits:
Forward-recovery plan:
Restore/recovery evidence:
Customer communication:
Cleanup and review date:
```

## Common Mistakes

- equating build success with readiness;
- using liveness as capability health;
- forgetting workers and scheduled processes;
- deploying incompatible schema or messages;
- assuming rollback reverses data and external effects;
- having backups without restore tests;
- omitting queue age, saturation, business invariants, or owners;
- leaving emergency controls in production.

## Exercises

1. Complete the release record for a mixed PHP-version deployment.
2. Design readiness and capability-health checks for a reservation service.
3. Plan a canary with database, queue, security, and user-impact gates.
4. Write forward recovery for an irreversible migration.
5. Run a tabletop incident using the artifact, observability, and communication checklist.
6. Create a retirement plan for compatibility flags and emergency credentials.

## Review Questions

- What does a green deploy fail to prove?
- How do liveness and readiness differ?
- Which boundaries make rollback unsafe?
- What must a restore test verify?
- Which business invariants belong on a production dashboard?
- Who owns promotion, abort, recovery, and cleanup?

## Summary

Production readiness covers artifact provenance, runtime compatibility, schema and message safety, configuration and secrets, health, traffic, workers, observability, capacity, rollout, rollback, forward recovery, restore evidence, incident response, communication, and retirement. Treat the checklist as an owned evidence record, not a ceremony.

## Book Handoff

This chapter closes the reference volume and the book. The preceding chapters establish PHP language and runtime behavior, application boundaries, data structures, algorithms, databases, HTTP, security, testing, architecture, performance, distributed systems, legacy migration, engineering judgment, and operational readiness. Keep the continuation state aligned with the repository and revisit version-sensitive claims as the supported fleet changes.

## References

- [Chapter 254 — Linux for PHP Engineers](../17-production-engineering/254-linux-for-php-engineers.md)
- [Chapter 260 — Logging](../17-production-engineering/260-logging.md)
- [Chapter 263 — Health Checks](../17-production-engineering/263-health-checks.md)
- [Chapter 264 — Deployment](../17-production-engineering/264-deployment.md)
- [Chapter 265 — Rollback](../17-production-engineering/265-rollback.md)
- [Chapter 267 — Backups](../17-production-engineering/267-backups.md)
- [Chapter 268 — Disaster Recovery](../17-production-engineering/268-disaster-recovery.md)
- [Chapter 294 — Incident Investigation](../20-senior-engineering/294-incident-investigation.md)
