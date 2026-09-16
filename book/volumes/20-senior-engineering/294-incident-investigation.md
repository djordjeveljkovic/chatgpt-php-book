---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 294
title: Incident Investigation
slug: incident-investigation
status: complete
summary: ../../_ai/chapter-summaries/294-incident-investigation-summary.md
---

# Chapter 294 — Incident Investigation

## Why This Matters

An incident investigation is a controlled effort to reduce harm, preserve trustworthy evidence, understand scope, recover safely, and improve the system without blaming individuals for systemic conditions. An event becomes an incident when its impact or uncertainty requires coordinated action.

Not every incident is a security incident, but every security incident needs operational discipline. Distinguish event from incident, symptom from impact, containment from recovery, investigation from remediation, and confirmed fact from hypothesis.

## Declare and Classify

Create an initial record with an incident ID, start and detection times, declaring person, affected capability, known and suspected impact, severity, operational/security classification, current containment, incident commander, and next update time. Classification may change as evidence develops.

Classify consequences by availability, correctness, data integrity, confidentiality, authorization, credential compromise, regulatory or contractual exposure, financial impact, and safety. Record uncertainty instead of downgrading a problem because the affected set is not yet known.

## Establish Incident Command

Use lightweight roles:

| Role | Responsibility |
| --- | --- |
| incident commander | prioritizes decisions and coordinates response |
| operations lead | executes technical containment and recovery |
| investigation lead | maintains hypotheses, evidence, and timeline |
| communications lead | coordinates stakeholder updates |
| security/privacy lead | handles compromise and reporting obligations |
| scribe | records actions, timestamps, owners, and results |

The commander coordinates; they do not need to perform every technical action. Explicit authority prevents several responders from changing unrelated variables while nobody records the result.

## Stabilize, Contain, Preserve

Run two tracks:

~~~text
reduce ongoing harm       preserve investigation quality
  stop unsafe effects       capture evidence
  isolate affected scope    record actions
  protect users and data    maintain a reliable timeline
~~~

Containment may pause a deployment, disable a feature, stop a queue producer, quarantine a message, revoke a credential, isolate a tenant or region, restrict an administrative path, switch to read-only mode, or rate-limit an abusive endpoint. Every action needs expected benefit, risk, scope, owner, timestamp, and verification.

Preserve deployment/configuration history, feature flags, PHP-FPM/web/application logs, database audit and query evidence, queue payloads and retry metadata, identity logs, provider request IDs, affected objects, and relevant process state. For each item record an evidence ID, source, custodian, collection time and method, scope, integrity marker, storage location, access history, sensitivity, retention, and analysis notes. This evidence register makes provenance reviewable; it does not automatically satisfy formal forensic requirements.

Restarts, log rotation, queue purges, cache flushes, credential deletion, and cleanup scripts may destroy evidence. Coordinate sensitive collection with security, privacy, legal, or forensic specialists.

## Build a Factual Timeline

Use one timezone, preferably UTC. Record time, fact or action, source, confidence, and owner. Separate occurrence, detection, ingestion, deployment, containment, recovery, and delayed-side-effect times. Command completion is not desired state. Correlation is not causation. Gaps should remain marked as unknown rather than filled with a convenient story.

## Scope Customer and Data Impact

Ask which customers, tenants, users, regions, releases, requests, jobs, records, files, or messages were involved. Determine whether data was unavailable, altered, duplicated, exposed, or deleted; whether the affected set is known, bounded, or unknown; and what evidence supports the estimate.

Separate confirmed, probable, possible, ruled-out, and unknown impact. For tenant exposure record source tenant, possible recipient, resource and sensitivity, access method, time window, confirmed or inferred access, containment, and notification decision. Check queries, cache keys, search projections, queues, exports, object storage, support tools, logs, backups, and policy versions.

## Credential Compromise

Identify the credential, owner, privileges, systems and time window. Preserve access evidence and revoke compromised credentials promptly. Use overlap only when operationally necessary and demonstrably safe; then identify sessions/workers/jobs/integrations using the old value, inspect reads/writes/exports/configuration changes, reconcile external effects, and record residual uncertainty. This applies to database keys, object storage, Composer/CI tokens, webhook and signing keys, sessions, reset tokens, provider credentials, and operator accounts. Removing a value from Git history does not revoke it.

## PHP, Database, and Queue Investigation

Correlate PHP-FPM artifact versions with failures, worker saturation, memory, reloads, proxy timing, and FPM/CLI configuration. Distinguish application errors from upstream timeouts and ask whether an external effect completed after the client timed out.

For databases inspect locks, waits, connections, replication, migrations, transaction duration, audit records, and whether writes committed, rolled back, or remain uncertain. Avoid ad hoc production updates during investigation.

For queues inspect backlog age, retry counts, leases, poison messages, dead-letter state, stale workers, and message-version compatibility. Determine whether work never started, partially completed, completed externally, or was acknowledged incorrectly. Never purge messages merely to make a queue look healthy.

## Rollback, Forward Recovery, Reconciliation

Rollback is a decision with preconditions. Evaluate compatibility across application versions, schema, messages, caches, projections, credentials, external effects, and workers. Compare code/configuration rollback, feature disablement, traffic reduction, queue pause/drain, data repair, replay, forward migration, and compensating actions.

Forward recovery may be safer when schema changes, messages, provider effects, or credentials cannot be undone. Recovery is incomplete until service health, customer-visible behavior, data integrity, queue convergence, duplicate-effect checks, and continued monitoring are verified.

## Privacy, Legal, and Communication Boundaries

Engineers should not independently make legal conclusions. Involve security, privacy/data-protection, legal, compliance, communications, contractual account owners, and affected vendors when the data, jurisdiction, contract, or notification obligation requires it. Keep factual engineering records separate from speculative conclusions and record who made notification decisions and what evidence supported them.

Responders need current facts, hypotheses, owners, commands, results, and next decision time. Leaders need impact, business risk, mitigation, uncertainty, and authority decisions. Customers need affected capability, time window, workaround, data or transaction implications, and next update. Use “we observed,” “we suspect,” “we have not determined,” and “we verified” precisely.

## Blameless Learning

Blameless does not mean consequence-free or evidence-free. Examine system design, incentives, deadlines, missing guardrails, ownership, alert quality, deployment and rollback assumptions, training, operational load, and communication. “Human error” is not a complete cause; ask what made the error likely and recovery difficult.

Follow-ups should be specific, owned, prioritized, bounded, and verifiable. Classify them as containment, corrective fix, detection, resilience, security, documentation, technical debt, or architecture work. Do not close the incident merely because traffic recovered.

## Case Study

In a tenant-scoped PHP catalog, a deployment introduces an authorization regression, some cache keys omit tenant scope, FPM errors rise while workers continue processing, a database credential appears in diagnostics, notifications time out with unknown completion, and support reports cross-tenant results.

Declare and classify the incident, assign roles, preserve evidence, contain the feature, rotate credentials, bound tenant and time-window impact, inspect database/cache/FPM/queue evidence, choose rollback or forward recovery, coordinate privacy and customer communication, and write corrective actions with verification dates.

## Common Mistakes

* restarting before collecting evidence when safety did not require it;
* blaming the latest deploy without comparing cohorts and alternatives;
* calling an incident resolved when errors fall but data or queues remain unreconciled;
* purging queues/caches or editing production data during diagnosis;
* rotating credentials without inventorying workers, sessions, and integrations;
* treating possible exposure as confirmed exposure;
* retrying external effects without reconciliation;
* treating “human error” as the root cause;
* publishing speculative customer or legal claims;
* performing forensic work without authorization;
* assigning follow-ups without owners, dates, or verification.

## Senior Engineer Thinking

Strong incident investigation combines command, evidence, containment, impact analysis, safe recovery, careful communication, and durable learning. It protects customers while preserving the information needed to improve the system.

## Exercises

1. Write an incident declaration and severity decision.
2. Build an evidence register and factual timeline.
3. Classify confirmed, probable, possible, ruled-out, and unknown customer impact.
4. Design a credential-compromise response plan.
5. Analyze tenant exposure across queries, caches, jobs, exports, and logs.
6. Map PHP-FPM, database, and queue evidence.
7. Choose rollback, forward recovery, containment, or reconciliation.
8. Create a privacy/legal escalation plan.
9. Write a blameless post-incident review with corrective actions.
10. Draft internal and customer updates under uncertainty.

## Review Questions

* When should an event become an incident?
* What must be preserved before containment?
* How do you distinguish possible exposure from confirmed access?
* Why is credential rotation both containment and a new compatibility risk?
* When is rollback unsafe?
* What evidence assesses tenant exposure?
* How should unknown outcomes be handled?
* When should privacy or legal specialists be involved?
* What makes a post-incident action effective?
* How can an investigation remain blameless and rigorous?

## Summary

Incident investigation reduces harm, preserves trustworthy evidence, establishes scope, coordinates roles, contains safely, determines customer and data impact, handles credentials and tenant exposure, chooses compatible recovery, communicates uncertainty, and converts learning into owned corrective work.

## References

- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md)
- [Chapter 291 — Debugging Production](291-debugging-production.md)
- [Chapter 293 — Security Review](293-security-review.md)

## Chapter 295 Handoff

Incidents leave teams with incomplete evidence, competing priorities, and consequential choices. Chapter 295 will examine how senior engineers make and record technical decisions under uncertainty, including trade-offs, reversibility, risk, and accountability.
