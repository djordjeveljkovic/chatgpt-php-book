---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 265
title: Rollback
slug: rollback
status: complete
summary: ../../_ai/chapter-summaries/265-rollback-summary.md
---

# Chapter 265 — Rollback

## Why This Matters

Rollback is the controlled restoration of a previously serving runtime revision when the current revision is unsafe, unhealthy, or incompatible. It is not the same as pressing an undo button. Code, configuration, feature flags, database schema, persisted data, queue messages, credentials, and external effects do not necessarily move together or in reverse.

A rollback can fail even when the previous application artifact is available. The old code may not understand a new column, a new message, a changed serialized value, or a provider response. A secret may already have been revoked. An email may already have been sent. A migration may have changed data in a way that an old release cannot safely consume.

Treat rollback as a prepared production capability. Know what can be restored, what must be repaired or rolled forward, which compatibility window remains open, and what evidence proves that the restored population is safe.

## Mental Model

A deployment changes several dimensions at once:

~~~text
code and dependencies
configuration and secrets
schema and persisted data
messages and workers
traffic and feature exposure
external side effects
        ↓
observe, classify, contain
        ↓
restore compatible runtime | repair state | roll forward | accept risk
~~~

Rollback is usually a new deployment of an older artifact. It still needs startup checks, readiness gates, capacity, traffic control, observation, and an owner. The fact that a platform can select an earlier image or release directory does not prove that the application can safely serve the current state.

Keep three questions separate:

1. Can the previous artifact start?
2. Can it safely read and write the current state?
3. Will restoring it reduce customer harm without losing evidence or repeating effects?

The second and third questions are the hard ones.

## Rollback Is Multidimensional

Record the state of each rollback dimension before changing production:

| Dimension | What may have changed | Possible recovery |
| --- | --- | --- |
| Code | PHP classes, dependencies, generated assets | Re-deploy a verified artifact |
| Configuration | URLs, limits, defaults, behavior switches | Restore a versioned configuration |
| Secrets | Key IDs, credentials, certificates | Re-enable compatible credentials or rotate safely |
| Schema | Columns, indexes, constraints, migrations | Keep compatibility, repair, or restore a database |
| Data | Rows, values, deletion, corruption | Compensating repair or point-in-time recovery |
| Messages | New payloads, retries, acknowledgments | Keep consumers compatible, quarantine, or replay |
| Traffic | Routes, cohorts, regions, percentages | Shift traffic to a known-safe population |
| External effects | Emails, payments, webhooks, provider calls | Reconcile; do not pretend they were undone |

Do not use “rollback completed” for only one row of this table. Say which dimensions were restored and which remain changed.

## Rollback, Roll-Forward, and Containment

Choose the response from the failure, not from habit.

### Rollback

Restore an earlier compatible runtime when the previous revision can safely process current requests, state, and messages. This is useful for a bad code path, a missing extension, a wrong route, or a configuration error whose state remains compatible.

### Roll-forward

Deploy a new corrective revision when the old revision cannot understand the new state, when the defect is easier to fix forward, or when reverting would repeat or worsen an external effect. A small corrective release is often safer than forcing an incompatible old artifact back into service.

### Containment

Reduce exposure while preserving options. Pause the rollout, disable a feature flag, remove a route, shed noncritical work, stop a producer, quarantine messages, or restrict traffic to a safe cohort. Containment buys time; it is not a substitute for a durable recovery decision.

### Data recovery

Restore or repair persisted state only when the damage requires it and the recovery point is understood. Database restoration can lose valid writes after the chosen point, so preserve them for reconciliation before replacing state.

The safest response may combine these actions: disable a feature, stop new writes, deploy a compatible reader, reconcile effects, and then roll forward.

## Rollback Preconditions

Before a deployment begins, verify that rollback remains possible:

* the previous artifact is retained and independently verifiable;
* its PHP version, extensions, and runtime assumptions are available;
* the previous configuration and non-secret references are versioned;
* credentials and key IDs needed by the previous release remain valid;
* schema changes are additive or the old release has been tested against them;
* old workers can consume messages already emitted by the candidate;
* feature flags can be set to a safe state with an audited owner;
* capacity exists for the restored population and its drain overlap;
* deployment history identifies the exact revision, not only a mutable tag;
* operators have permission to change traffic and inspect evidence;
* the recovery path has been exercised in a production-shaped environment.

The precondition is not “we still have the old Docker tag.” It is “we can start a known artifact and safely process the state that exists now.”

## A Typed Rollback Gate

Make the decision explicit and conservative. An unknown signal should not silently become permission to proceed:

~~~php
<?php

declare(strict_types=1);

enum RecoveryAction: string
{
    case Observe = 'observe';
    case Contain = 'contain';
    case Rollback = 'rollback';
    case RollForward = 'roll-forward';
    case Repair = 'repair';
    case DataRecovery = 'data-recovery';
    case ManualIntervention = 'manual-intervention';
}

final readonly class RecoveryObservation
{
    public function __construct(
        public bool $candidateServing,
        public bool $previousArtifactAvailable,
        public bool $stateCompatible,
        public bool $telemetryFresh,
        public bool $externalEffectsReconciled,
        public float $errorRatio,
    ) {
        if (!is_finite($errorRatio) || $errorRatio < 0 || $errorRatio > 1) {
            throw new InvalidArgumentException('Invalid error ratio');
        }
    }
}

function chooseRecovery(
    RecoveryObservation $observation,
    float $rollbackErrorRatio,
): RecoveryAction {
    if (!is_finite($rollbackErrorRatio) || $rollbackErrorRatio < 0 || $rollbackErrorRatio > 1) {
        throw new InvalidArgumentException('Invalid rollback threshold');
    }

    if (!$observation->telemetryFresh) {
        return RecoveryAction::Contain;
    }

    if ($observation->candidateServing && $observation->errorRatio <= $rollbackErrorRatio) {
        return RecoveryAction::Observe;
    }

    if (!$observation->previousArtifactAvailable || !$observation->stateCompatible) {
        return RecoveryAction::RollForward;
    }

    if (!$observation->externalEffectsReconciled) {
        return RecoveryAction::Contain;
    }

    return RecoveryAction::Rollback;
}
~~~

This policy intentionally does not attempt to infer business safety from a single error ratio. The example returns only the gate's common automatic actions; repair, data recovery, and manual intervention remain explicit outcomes for a fuller policy. A real implementation needs observation windows, sample counts, affected cohorts, migration state, message compatibility, action ownership, and an idempotent controller. The policy can say “rollback is allowed”; the mechanism must still execute and verify the transition.

## A Rollback Sequence

Use a bounded sequence with explicit evidence:

~~~text
declare incident and recovery owner
        ↓
pause promotion and preserve evidence
        ↓
contain exposure and stop unsafe producers
        ↓
classify code, state, message, and effect compatibility
        ↓
select rollback, roll-forward, repair, or data recovery
        ↓
prepare exact artifact, configuration, credentials, and capacity
        ↓
start and verify the recovery population
        ↓
shift traffic or work gradually
        ↓
observe service and business outcomes
        ↓
reconcile durable and external effects
        ↓
close the recovery with a recorded state
~~~

Do not delete the failing artifact or logs before investigation. Preserve the candidate image, release manifest, configuration version, deployment events, traces, queue state, migration output, and operator actions according to the retention policy.

## Runtime Rollback: PHP-FPM, Nginx, and Containers

For a release-directory deployment, select the previous immutable directory and reload the serving processes according to the drain contract from Chapters 255, 256, and 264. Do not copy old files over the current directory while requests are executing. Verify the effective release from the externally visible path, not just from a symlink or container specification.

OPcache state and process-local state are separate concerns. With unique release paths, new requests resolve the selected path and compile it; with reused paths or timestamp validation disabled, apply the configured OPcache reset or process-reload policy explicitly. Existing PHP-FPM workers can still have compiled scripts from the previous release, while long-running CLI workers can retain application state in memory. Drain and recycle both populations according to configured deadlines. A forced termination can cause redelivery or an ambiguous external result, so recovery must reconcile the resulting operation records.

Container rollback means selecting a verified immutable image or release digest. A mutable tag can point to different bytes at different times. Keep runtime configuration, secrets, volumes, and migrations as separate rollback dimensions. Reverting an image does not revert a mounted volume or database.

Nginx can reload a prior valid configuration, but a successful reload does not prove that the intended PHP-FPM socket, release path, proxy policy, or health route is active. Verify the complete request path after traffic changes.

## Database and Schema State

The most important rollback rule is: application code can usually be replaced faster than data can be reversed.

An additive migration often keeps rollback open:

~~~text
add nullable column
        ↓
deploy code that reads old and new forms
        ↓
backfill or dual-write with observation
        ↓
switch reads and writes
        ↓
remove old form only after rollback window expires
~~~

If the candidate only adds a compatible column, the previous code may continue to work. If the candidate changes the meaning of an existing value, deletes data, tightens a constraint, changes an enum, or removes a column, test the previous release against the resulting state before declaring rollback safe.

Do not automatically run a down migration during an application rollback. A down migration can destroy data written by the candidate, conflict with newer readers, or take locks during an incident. Prefer a forward-compatible repair migration or a carefully reviewed point-in-time restore when the data itself is damaged.

Database restoration has a scope and a cut-off. A full backup plus transaction-log or binary-log replay can recover to a chosen point for engines and tooling that support it, but changes after that point must be captured and reconciled. A database transaction rollback only affects the transaction whose changes have not committed; it cannot retract an email or provider call that already happened.

## Messages, Workers, and External Effects

Messages outlive deployments. If the candidate emitted a new message version, the previous consumer must either understand it or the message must be transformed, quarantined, or consumed by a compatible adapter. Do not purge a queue as a shortcut unless loss is explicitly acceptable and recorded.

For a worker rollback:

1. stop the candidate from claiming new work;
2. let in-flight work finish within a deadline where safe;
3. acknowledge only after the durable effect succeeds;
4. release or negatively acknowledge unfinished work for redelivery;
5. use operation identity and idempotency to handle ambiguous completion;
6. start compatible consumers before resuming normal intake;
7. measure queue age, redelivery, failure, and reconciliation progress.

External effects require reconciliation, not fiction. A payment authorization, email, webhook, inventory reservation, or third-party mutation may have succeeded even when the candidate reported failure. Before retrying after rollback, inspect durable operation evidence and provider status. A compensating action must have its own authorization, idempotency key, audit record, and business policy.

## Feature Flags, Configuration, and Secrets

A feature flag can contain exposure without changing the deployed code. Disable the flag when the old and new code both understand its state and the disabled path is safe. If the candidate has already written data that the old code cannot interpret, the flag alone cannot make rollback safe.

Restore configuration as a versioned set with an explicit effective timestamp. Do not blindly restore every value from the earlier release: a rotated certificate, endpoint, or security policy may need to remain current. Separate code compatibility from credential validity.

Secrets need overlap. Keep old verification keys or credentials available until all candidate workers, delayed jobs, cached requests, and rollback paths no longer require them. Conversely, revoke a compromised secret promptly and choose a compatible corrective release rather than restoring code that depends on it.

## Partial Rollback and Mixed Populations

Rollback is often gradual. During the transition, old candidate instances, restored instances, workers, queues, caches, and clients may coexist. Track the serving population by release and role. A platform’s “rolled back” event may describe only the desired template; instances may still be starting, draining, or unavailable.

If only one region or zone is restored, compare its traffic, dependency behavior, schema state, and business outcomes with the unrecovered population. Do not call the incident resolved because one canary is healthy. If the restored release performs worse, stop progression and choose a corrective roll-forward or containment plan.

## Observability and Evidence

Record recovery events with:

* incident and operation identifiers;
* candidate and selected release identities;
* configuration, flag, secret-key, and migration versions;
* start, contain, shift, drain, and completion timestamps;
* affected routes, tenants, regions, and worker roles;
* error, latency, queue, saturation, and restart signals;
* durable-effect and reconciliation counts;
* operator, controller, and approval identity;
* unresolved unknowns and next review time.

Metrics show prevalence; logs and traces help explain representative failures; durable records prove business state. Keep the recovery identity bounded so observability does not create an unmanageable metric series. Missing telemetry is an unknown state and should block an automatic “healthy” conclusion.

## Performance and Capacity

A rollback consumes capacity just like a deployment. The restored population may require different PHP extensions, connection behavior, cache warm-up, query plans, or worker limits. Keep enough capacity to run candidate and restored workers while draining, and reserve room for a second failure.

Recovery speed is not the only objective. A rapid traffic switch can amplify a bad assumption; a slow switch can prolong customer harm. Set a rate from detection time, compatibility confidence, drain deadlines, dependency headroom, and the cost of another incorrect effect.

Database restore and reconciliation can create heavy I/O, lock contention, queue replay, and provider traffic. Throttle repair and replay. A recovery that overloads the healthy system can become a second incident.

## Security

Rollback artifacts are sensitive. They may contain vulnerable dependencies, old configuration references, debugging tools, or credentials that were valid only temporarily. Retain them under access control, verify provenance before use, and record who selected them.

Do not let an emergency bypass artifact signatures, review boundaries, authorization, or audit records without an explicit incident policy. Emergency access should be narrow, time-limited, and reviewed afterward. A rollback command with broad production permissions is a high-impact control plane.

When restoring data, protect backup and transaction-log access as production data access. Redact recovery logs, avoid exposing secret values in manifests, and ensure that restored copies do not become an untracked source of personal or financial information.

## Concurrency and Idempotency

Only one recovery controller should own a service and environment at a time. Use a durable lease or compare-and-set state transition. A retry after an ambiguous response must converge on the declared target release rather than apply “one more” traffic shift or migration.

Persist a recovery record with a stable operation identity:

~~~text
recovery_id
candidate_release
target_release
state_version
current_stage
traffic_scope
schema_version
message_contracts
effect_reconciliation_state
last_observation
owner_lease
~~~

Before each action, read the current state and verify that the expected predecessor still holds. After the action, record the observed result. If the controller loses contact, a new controller can resume or safely stop without guessing which steps succeeded.

## Testing Rollback

Test the recovery path before an incident:

* restore a previous PHP artifact against the pre-expand and expanded schemas when partial migration is possible;
* test code, configuration, feature-flag, and secret-key overlap;
* interrupt a deployment before traffic, during canary traffic, and during drain;
* force-terminate requests and workers with ambiguous completion;
* publish old and candidate messages, then deliver each to previous, candidate, and corrective consumers through the supported overlap;
* simulate revoked credentials and unavailable artifact storage;
* verify that a mutable tag cannot masquerade as a known artifact;
* restore a database copy and replay logs to a declared recovery point;
* capture writes after that point for reconciliation;
* inject missing metrics, stale traces, and a lost controller lease;
* retry every recovery action after a timeout or lost response;
* rehearse containment, roll-forward, and data-repair decisions when rollback is unsafe.

A tabletop exercise should name the person who can authorize each action, the evidence needed to proceed, the maximum tolerated data loss, and the exact point at which the team stops automated recovery.

## Common Mistakes

* Assuming an old image or symlink means the old system is safe.
* Treating application rollback as database rollback.
* Running destructive down migrations automatically during an incident.
* Restoring code that cannot read current schema, messages, or serialized state.
* Revoking secrets before delayed jobs and rollback workers have migrated.
* Replaying provider calls without checking operation evidence.
* Purging queues or backups to make the incident look complete.
* Letting multiple recovery jobs shift traffic concurrently.
* Treating a platform “rollback succeeded” event as customer recovery.
* Ignoring capacity, cache warm-up, database load, or queue replay pressure.
* Losing the candidate artifact and logs before root-cause analysis.
* Bypassing provenance, authorization, or audit controls because the change is urgent.

## Senior Engineer Thinking

The senior question is not “can we return to the previous commit?” It is “which state does that commit expect, which effects have already escaped, and which recovery action reduces harm while preserving correctness and evidence?”

Rollback readiness is a property of the whole system: artifacts, contracts, schema evolution, message compatibility, credentials, traffic control, capacity, observability, and operator practice. A reliable team knows when rollback is safe, when containment is safer, and when a corrective roll-forward is the only honest path.

## Exercises

1. Create a rollback matrix for a release that adds a column, changes a queue message, rotates a signing key, and enables a feature flag. Mark each dimension as reversible, compatible, repairable, or irreversible.
2. Extend chooseRecovery with minimum sample counts, an observation window, and an explicit action for unknown schema compatibility.
3. Design a worker rollback drill that forces termination after a provider call but before acknowledgment. Specify the durable evidence and idempotency check that prevent a duplicate effect.
4. Plan point-in-time database recovery after an accidental delete. Identify the recovery cut-off, writes to preserve, reconciliation owner, and validation queries.

## Review Questions

* Why is rollback usually a new deployment rather than an undo operation?
* Which dimensions can change independently of application code?
* When is roll-forward safer than rollback?
* Why can an additive schema change preserve rollback while a destructive change closes it?
* Why should a down migration not run automatically during an incident?
* How should workers acknowledge, release, and retry work during rollback?
* Why must external effects be reconciled rather than assumed to have been undone?
* What does a platform rollback-complete event fail to prove?
* Why do recovery actions need leases, compare-and-set transitions, and idempotency?
* What evidence must be preserved before deleting or replacing a failing artifact?

## Summary

Rollback restores a compatible runtime revision; it does not automatically reverse schema changes, persisted writes, queue messages, credentials, or external effects. Prepare rollback before deployment by retaining verifiable artifacts, compatibility, capacity, secrets, flags, evidence, and an exercised recovery path. During an incident, contain exposure, classify state and effect compatibility, choose rollback, roll-forward, repair, or data recovery deliberately, execute through an idempotent controller, and verify customer and business outcomes. A recovery is complete only when its remaining unknowns and irreversible effects are recorded and owned.

## References

- [Kubernetes: Deployments and rollback history](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes: Rolling update and rollback](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [Nginx: Controlling nginx](https://nginx.org/en/docs/control.html)
- [PHP Manual: OPcache configuration](https://www.php.net/manual/en/opcache.configuration.php)
- [PHP Manual: FastCGI Process Manager](https://www.php.net/manual/en/install.fpm.php)
- [MySQL 8.0 Reference Manual: Backup and recovery types](https://dev.mysql.com/doc/refman/8.0/en/backup-types.html)
- [MySQL 8.4 Reference Manual: Point-in-time recovery](https://dev.mysql.com/doc/refman/8.4/en/point-in-time-recovery.html)
- [PostgreSQL: ROLLBACK](https://www.postgresql.org/docs/current/sql-rollback.html)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 245 — Dead-Letter Queues](../16-distributed-systems/245-dead-letter-queues.md)
- [Chapter 255 — Nginx](./255-nginx.md)
- [Chapter 256 — PHP-FPM](./256-php-fpm.md)
- [Chapter 257 — Containers](./257-containers.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 262 — Tracing](./262-tracing.md)
- [Chapter 263 — Health Checks](./263-health-checks.md)
- [Chapter 264 — Deployment](./264-deployment.md)
- [Chapter 266 — CI/CD](./266-ci-cd.md)
- [Chapter 277 — Database Migration](../18-legacy-php/277-database-migration.md)
