---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 291
title: Debugging Production
slug: debugging-production
status: complete
summary: ../../_ai/chapter-summaries/291-debugging-production-summary.md
---

# Chapter 291 — Debugging Production

## Why This Matters

A production incident is not a puzzle to solve from intuition. It is a live system state that must be stabilized, measured, and reasoned about without destroying the evidence needed to find the cause. Real traffic, tenant behavior, retries, deployment overlap, dependency failure, and irreversible effects make production different from a local reproduction.

The first objective is to reduce customer and data harm while preserving the system’s ability to provide evidence. Root-cause certainty comes later. A fast but unrecorded restart may improve a graph while erasing the process state, queue lease, or memory pattern that explains the failure.

## Stabilize Before Diagnosing

Run two tracks deliberately:

~~~text
stabilize the service
        ├── reduce customer and data harm
        └── preserve evidence
investigate the cause
        ├── collect observations
        └── test bounded hypotheses
~~~

Possible containment actions include pausing a rollout, disabling a feature flag, rate-limiting an expensive endpoint, stopping a queue producer, quarantining a poison message, switching to read-only or degraded mode, isolating a tenant or worker cohort, fencing a stale writer, or rolling forward when compatibility permits it.

For every action record its scope, expected benefit, risk, duration, verification, owner, and abort condition. A mitigation is an experiment with consequences, not a free change because the incident is urgent.

## Symptoms Are Not Causes

An observation narrows a search; it does not identify a cause:

| Observation | Possible causes |
| --- | --- |
| HTTP 500 increase | code defect, dependency failure, configuration, database error |
| PHP-FPM pool full | slow SQL, provider timeout, CPU saturation, memory pressure |
| queue backlog | slow consumer, poison messages, retry storm, producer surge |
| cache-miss spike | eviction, key-version change, outage, deployment behavior |
| elevated latency | lock contention, network delay, application regression, saturation |

Use the chain:

~~~text
symptom → affected boundary → resource pressure → triggering change → underlying cause
~~~

“Redis is down” may be a confirmed observation, but it may not explain why a request path has no bounded fallback or why a queue is now retrying every message. “The latest deploy caused it” is a hypothesis until timing, cohorts, artifact identity, and rollback behavior support it.

## Define Impact and Scope

Before changing anything, establish:

* affected capability and user-visible behavior;
* start time, detection time, and current trend;
* affected tenants, routes, regions, releases, and worker cohorts;
* error rate, latency distribution, queue age, and resource pressure;
* data-integrity, security, and external-effect implications;
* a known-good comparison;
* what remains healthy;
* whether missing telemetry could hide additional impact.

Distinguish “zero failures observed” from “failure data unavailable.” A quiet dashboard can mean no traffic, dropped telemetry, a broken exporter, or a query that excludes the affected cohort.

## Preserve Evidence

Create an incident evidence packet containing an incident ID, declaration time, observed symptom, affected capability, release and artifact, configuration and feature-flag versions, schema and migration versions, request/trace/operation IDs, logs, metrics, traces, PHP-FPM and host state, database and cache state, queue depth and retry data, commands executed, changes made, observed results, and open hypotheses.

Preserve deployment and configuration history, feature-flag changes, application/Nginx/FPM logs, metrics and traces, queue messages and retry metadata, database wait and lock information, provider responses, host/container/process state, and security audit events. Restarts, log rotation, cache clearing, queue purges, credential rotation, and cleanup commands can destroy evidence or alter the condition being measured.

Security-sensitive incidents need the security owner’s direction before collecting or modifying sensitive systems. Record access and protect incident data; debugging authority does not override privacy or least-privilege requirements.

## Build a Factual Timeline

Use one timezone, preferably UTC, and separate event time from ingestion time:

| Time | Observation | Action | Result | Confidence |
| --- | --- | --- | --- | --- |
| 14:05 | 5xx rate increased | none | errors continued | high |
| 14:08 | release deployed | paused rollout | new instances stopped | high |
| 14:12 | database lock wait increased | reduced write traffic | queue growth slowed | medium |

Track first occurrence, first detection, alert creation, deployment start and completion, configuration changes, mitigation, recovery, and delayed side effects. Record uncertainty explicitly. Command completion is not the same as desired state, and correlation is not causation.

## Use Hypothesis Loops

For each hypothesis:

1. state the claim;
2. identify a predicted signal;
3. select the smallest safe observation or experiment;
4. gather evidence across independent signals;
5. update confidence;
6. choose the next action.

Example: if the new release reads a nullable column before the migration reaches all instances, failures should cluster by artifact and schema version. Compare outcomes by release, schema, route, and payload shape. If only old-schema instances fail, pause rollout and inspect compatibility.

A useful hypothesis explains why the symptom appeared at that time, in that scope, and with that shape. A check that cannot distinguish competing explanations is observation, not an experiment.

## Observability as Evidence

Use the right signal for the question:

| Signal | Best evidence |
| --- | --- |
| metrics | aggregate rates, distributions, saturation, and trends |
| logs | detailed events, decisions, errors, and structured context |
| traces | request paths and dependency timing across boundaries |
| health checks | readiness, drain state, and dependency policy |
| audit records | durable business or security actions |

Correlate a request or operation through its boundaries: request ID, trace, PHP-FPM worker, SQL query or lock, cache operation, provider call, queue message, and durable business record. Use deployment version, tenant-safe identifiers, queue message ID, operation identity, and outcome fields where they are bounded and safe. Sampling, clock skew, missing instrumentation, high-cardinality labels, delayed telemetry, exporter failure, secrets, and personal data limit what can be concluded.

## PHP-FPM and HTTP Request Debugging

Inspect active, idle, and maxed-out FPM workers, listen-queue growth, slow logs, worker memory and restarts, Nginx upstream timing, request and dependency deadlines, OPcache or artifact identity, graceful reload behavior, and extension or SAPI configuration differences.

Ask:

* Is the request waiting for a worker?
* Is a worker occupied by SQL, a provider, a lock, or CPU?
* Did only one release or pool change?
* Does the request finish after the client timeout?
* Could an external effect have happened despite an error response?

A full pool is a symptom, not proof that FPM itself is defective. Distinguish PHP limits such as `memory_limit` and `max_execution_time` from web-server, proxy, database, container, and file-descriptor limits. A local timeout may leave the database or provider processing.

For a bounded diagnostic sample, application code can record process-local memory without exposing request data:

~~~php
<?php

function memoryObservation(): array
{
    return [
        'current_bytes' => memory_get_usage(true),
        'peak_bytes' => memory_get_peak_usage(true),
    ];
}
~~~

This measures the PHP process’s view at one point; it does not prove container usage, total worker health, or a leak. Keep the diagnostic flag scoped, sampled, access-controlled, and temporary.

## CLI Commands and Long-Running Workers

CLI and worker debugging differs from FPM because process state persists across messages. Memory can grow, stale configuration can remain loaded, connections and transactions need explicit cleanup, one poison message can block progress, retries can amplify load, and graceful shutdown can leave in-flight work ambiguous.

Inspect worker age and release identity, current message and lease, queue age and retry count, memory trend, signal handling, idempotency and acknowledgement timing, and failed-message or dead-letter state. Never delete a queue or acknowledge messages merely to make backlog metrics look healthy.

## Databases, Caches, and Queues

| Boundary | Evidence | Safe questions |
| --- | --- | --- |
| database | locks, waits, plans, connections, replication | is work blocked, slow, rejected, or reading stale data? |
| cache | hit rate, latency, evictions, key version, memory | is cache optional, stale, unavailable, or authoritative by mistake? |
| queue | depth, oldest age, retries, leases, consumer rate | is production faster than consumption, or repeatedly failing? |
| provider | timeout, status, request ID, quota | is the outcome known, unknown, or safely retryable? |

Check tenant scope, noisy neighbors, connection exhaustion, retry storms, cache stampedes, and unbounded work. Do not perform ad hoc production updates during diagnosis unless explicitly authorized, reviewed, bounded, logged, and reversible.

## Unknown Completion and Safe Retries

A timeout does not prove non-completion. For every uncertain command:

1. identify its operation or idempotency key;
2. query durable local state;
3. inspect provider, queue, or audit evidence;
4. reconcile the ambiguous outcome;
5. retry only when duplicate effects are safe.

This applies to payments, reservations, notifications, imports, and webhooks. A client that received an error may still have caused a durable reservation or external provider action. Retrying blindly can create duplicates while making the incident harder to measure.

## Safe Production Experiments

Classify experiments by reversibility and blast radius:

* read-only inspection;
* sampled telemetry;
* one-tenant or one-cohort change;
* feature-flag adjustment;
* traffic reduction;
* worker restart or drain;
* rollback or roll-forward;
* data repair or replay.

Before each experiment state the question, expected observation, scope, risk, duration, abort condition, verification, and owner. Prefer an experiment that distinguishes hypotheses. Avoid uncontrolled restarts, arbitrary configuration changes, production data edits, cache flushes, and repeated retries without a stopping condition.

## Recovery and Verification

Recovery may involve stopping a rollout, rollback, roll-forward, feature disablement, queue throttling, provider failover, data restoration, projection rebuilding, external-effect reconciliation, or forward recovery when a schema or message change cannot safely be undone.

Rollback is safe only when application versions, schemas, messages, caches, and external effects remain compatible. A Git revert can restore source while leaving transformed rows, emitted messages, populated caches, or sent notifications behind. Choose containment or forward recovery when reversal would create more harm.

Recovery is incomplete until verified through customer-visible success, error and latency trends, queue age and retry behavior, database integrity, cache or projection convergence, duplicate-effect checks, security review where relevant, and continued observation after mitigation. A successful command proves that the command ran; it does not prove that the system recovered.

## Communicate Uncertainty

Responders need current facts, active hypotheses, owners, commands and results, and the next decision time. Leadership needs impact, business risk, mitigation, forecast uncertainty, and decisions requiring authority. Support and customers need the affected capability, start time, current workaround, data or order implications, and next update time.

Do not claim root cause before confirmation. Do not expose credentials, personal data, or exploit details in incident channels. “We have confirmed” and “we currently suspect” are different statements and should remain different.

## Handoffs and the Investigation Record

A handoff should include current impact, timeline, confirmed facts, rejected and active hypotheses, recent changes, actions already taken, unsafe actions, next diagnostic step, recovery state, owners, and deadlines.

Close the investigation with the confirmed cause and contributing factors, evidence supporting the conclusion, customer and data impact, temporary containment, permanent corrective actions, observability gaps, technical-debt items, and follow-up owners and review dates. A post-incident record is useful when it preserves reasoning and changes the system, not when it merely assigns blame.

## Case Study: A Failing Tenant-Scoped Flow

Suppose a new release changes Chapter 287’s search path. Intermittent 500s appear, PHP-FPM workers become saturated, queue age rises because notifications are delayed, database locks appear during a retry path, and some clients receive timeout responses despite uncertain provider outcomes.

Investigate in this order:

1. stabilize the rollout and protect the affected tenants;
2. compare errors by release, route, tenant, and schema version;
3. correlate traces, FPM data, SQL waits, cache behavior, and queue messages;
4. test whether the bottleneck is query shape, connection pressure, lock contention, or provider latency;
5. determine whether writes and external effects completed;
6. choose bounded containment or a compatible recovery action;
7. verify durable state, projection convergence, and backlog drain;
8. document the incident and convert unresolved obligations into Chapter 290 debt records.

The investigation should not begin by flushing the cache, increasing every worker pool, or replaying every timeout. Each action must answer a question and preserve enough evidence for the next decision.

## Common Mistakes

* restarting before collecting evidence when safety did not require it;
* treating the first failing dependency as the root cause;
* changing several variables at once;
* relying on averages rather than distributions and saturation;
* confusing unavailable telemetry with zero failures;
* replaying unknown-outcome operations blindly;
* purging queues or caches to hide symptoms;
* debugging only the web request while ignoring workers and scheduled jobs;
* ignoring mixed releases and schema compatibility;
* assigning causation from correlation alone;
* declaring recovery when traffic returns but data remains unreconciled;
* failing to convert unresolved causes into owned technical-debt items.

## Senior Engineer Thinking

Production debugging is controlled learning under pressure. Stabilize harm, preserve evidence, define scope, narrow hypotheses, run bounded experiments, communicate uncertainty, recover deliberately, and verify both service behavior and durable data. The goal is not to look certain quickly; it is to make the next decision safer than the previous one.

## Exercises

1. Build a timeline from a fictional PHP-FPM saturation incident and mark confidence for each entry.
2. Separate symptoms, hypotheses, predictions, checks, and confirmed causes.
3. Write an evidence-preservation checklist for a queue failure.
4. Design a read-only diagnostic plan for database lock contention.
5. Investigate a cache-miss spike without flushing the cache.
6. Compare an FPM timeout with a provider timeout and identify unknown completion.
7. Create a safe experiment plan with scope, duration, verification, and abort conditions.
8. Write an incident handoff note for a mixed-version deployment.
9. Choose rollback, roll-forward, containment, or reconciliation for the case study.
10. Convert unresolved investigation findings into Chapter 290 debt records.

## Review Questions

* What must happen before diagnosis?
* Why is a symptom not a cause?
* Which evidence can a restart, cache flush, or queue purge destroy?
* How do you distinguish a full FPM pool from a slow dependency?
* Why are unknown provider outcomes dangerous to retry?
* What makes a hypothesis testable?
* How do metrics, logs, traces, health checks, and audit records complement one another?
* When is a queue backlog a symptom rather than the cause?
* What verifies recovery beyond a successful command?
* What belongs in the final investigation record?

## Summary

Production debugging is disciplined investigation of a live system. Stabilize users and data, preserve evidence, define impact, separate symptoms from causes, correlate metrics/logs/traces, inspect PHP and shared-resource limits, test bounded hypotheses, treat completion as unknown when necessary, choose compatible recovery actions, communicate uncertainty, and verify service and data recovery.

## References

- [Chapter 256 — PHP-FPM](../../volumes/17-production-engineering/256-php-fpm.md)
- [Chapter 260 — Logging](../../volumes/17-production-engineering/260-logging.md)
- [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md)
- [Chapter 262 — Tracing](../../volumes/17-production-engineering/262-tracing.md)
- [Chapter 263 — Health Checks](../../volumes/17-production-engineering/263-health-checks.md)
- [Chapter 265 — Rollback](../../volumes/17-production-engineering/265-rollback.md)
- [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md)
- [Chapter 280 — Tennis Reservation Service](../../volumes/19-small-engineering-projects/280-tennis-reservation-service.md)
- [Chapter 284 — Queue Worker](../../volumes/19-small-engineering-projects/284-queue-worker.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 288 — Code Review](288-code-review.md)
- [Chapter 289 — Architecture Review](289-architecture-review.md)
- [Chapter 290 — Technical Debt](290-technical-debt.md)

## Chapter 292 Handoff

Once an incident is stable enough to investigate, performance symptoms deserve their own measurement discipline. Chapter 292 will examine workload models, latency distributions, profiling, database and PHP bottlenecks, capacity limits, and performance experiments.

The next chapter should begin with:

> Performance problems are rarely solved by making one line of PHP faster. They are solved by measuring the workload, locating the constrained resource, and changing the bottleneck without violating correctness or operational limits.
