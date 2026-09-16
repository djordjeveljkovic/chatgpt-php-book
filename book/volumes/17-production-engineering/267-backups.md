---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 267
title: Backups
slug: backups
status: complete
summary: ../../_ai/chapter-summaries/267-backups-summary.md
---

# Chapter 267 — Backups

## Why This Matters

A backup is a retained copy or recovery representation of data from which a system can be restored. It is not automatically a recovery plan. A file can exist while being incomplete, corrupt, unreadable, encrypted with a lost key, too old for the business requirement, or impossible to restore within the available outage window.

PHP applications depend on more than one database. They use relational rows, object-storage files, queues, caches, search indexes, configuration, secrets, scheduled-job state, and deployment metadata. A database dump cannot restore an uploaded document. A source repository cannot restore a customer record. A queue backup may be harmful if replaying it repeats payments or emails.

Define the data boundary, recovery objective, owner, retention, protection, and restore test for every important state. The useful question is not “do we take backups?” It is “what loss and recovery time does this backup strategy actually provide, and what evidence proves it?”

## Mental Model

Backup design connects live state to a tested recovery point:

~~~text
live data and metadata
        ↓ capture consistently
backup representation
        ↓ protect and retain
independent recovery copy
        ↓ restore into isolation
validated recovery environment
        ↓ reconcile and promote
operational service
~~~

The backup is one part of the path. Capture, transfer, storage, encryption, retention, restore, validation, and promotion all have failure modes. A successful upload proves only that some bytes reached a destination.

## RPO and RTO

Two objectives frame a backup strategy:

* Recovery Point Objective (RPO): the target maximum age of a valid recovery point, which bounds time-based data loss. A domain-specific loss budget, such as an allowed number of unreconciled orders, should be identified separately.
* Recovery Time Objective (RTO): the maximum acceptable time to restore an agreed service level.

An RPO of one hour does not mean every record is recoverable to the last hour. It describes an objective that the capture and restore design must support; if the objective is missed, the actual recovery point may be older or unknown. An RTO for a read-only status page may differ from the RTO for order placement.

Write the objective per data class and capability. A daily logical dump may meet a low-value reporting RPO but fail the order ledger’s RPO. A highly current database backup may still fail the document-download capability if object storage has no independent copy.

## Identify What Must Be Recoverable

Create an inventory before choosing tools:

| State | Examples | Recovery question |
| --- | --- | --- |
| Primary database | rows, schema, indexes, permissions | Can the engine be restored consistently? |
| Change log | PostgreSQL WAL, MySQL binary log | Can changes be replayed to a declared point? |
| Object data | uploads, exports, media | Are bytes, metadata, and references aligned? |
| Queue state | pending work, leases, dead letters | Can work be replayed without duplicate effects? |
| Cache and search | derived keys, indexes, projections | Can they be rebuilt or safely discarded? |
| Configuration | release settings, routing, policies | Are versions and effective values recorded? |
| Secrets | encrypted references, key IDs | Can required material be recovered or rotated? |
| Deployment evidence | artifacts, manifests, migrations | Can the restored state be identified and operated? |

Classify each item as authoritative, derived, reproducible, or disposable. A derived search index may not need the same backup as an accounting ledger, but the rebuild time belongs in the RTO calculation.

## Backup Types

### Logical backups

A logical backup represents tables, rows, schema objects, or application records in a portable format. It can be selective and useful for extracting or migrating data, but restore time grows with the amount of data and the tool’s processing rate. It may also miss engine-level state, extensions, permissions, or large object details unless configured for them.

### Physical backups

A physical backup copies storage or database files in an engine-supported consistent form. It can restore large datasets faster, but it is usually tied to an engine, version, storage layout, or cluster boundary. Follow the selected database’s backup procedure; copying live files with an ordinary PHP filesystem function is not a portable consistency method.

### Continuous change capture

Write-ahead logs, binary logs, or an equivalent change stream can reduce the gap between base backups and a recovery point. The stream must be retained, complete, ordered as required by the engine, and protected with the base backup it depends on. A missing segment can make an apparently healthy chain unusable.

### Application-level exports

An application export can preserve a business representation, such as accounts, orders, or configuration. It is valuable for selective repair and reconciliation, but it may not preserve every constraint, index, audit record, binary object, or provider relationship. Keep it alongside, not instead of, engine-appropriate recovery where full restoration matters.

No type is universally best. Combine types according to data authority, RPO, RTO, portability, cost, and restoration risk.

## Consistency Boundaries

A backup must define what “same point” means. A database snapshot can be transactionally consistent for that database while an object uploaded five seconds later is absent. A queue export may race with acknowledgment. A configuration backup may not include a secret version used by the running release.

Choose and document a consistency boundary:

* database transaction or engine snapshot;
* coordinated snapshot across related stores;
* event or change-log position;
* application quiescence or write pause;
* explicitly tolerated skew with reconciliation.

Do not call a set of independently copied directories a system backup unless the recovery procedure accounts for their ordering and skew. Record capture time, source identity, log position, schema version, and included object ranges.

## A Typed Backup Record

Keep backup metadata durable and free of secret values:

~~~php
<?php

declare(strict_types=1);

enum BackupState: string
{
    case Started = 'started';
    case Complete = 'complete';
    case Verified = 'verified';
    case Failed = 'failed';
}

final readonly class BackupRecord
{
    public function __construct(
        public string $backupId,
        public string $dataSet,
        public string $capturedAt,
        public string $contentDigest,
        public ?string $sourcePosition,
        public BackupState $state,
        public int $bytes,
    ) {
        if ($backupId === '' || $dataSet === '' || $contentDigest === '') {
            throw new InvalidArgumentException('Backup identity is incomplete');
        }

        if ($bytes < 0) {
            throw new InvalidArgumentException('Backup size cannot be negative');
        }

        if (
            in_array($state, [BackupState::Complete, BackupState::Verified], true)
            && ($sourcePosition === null || $sourcePosition === '')
        ) {
            throw new InvalidArgumentException('Completed backup needs a source position');
        }
    }
}

function canUseForRestore(BackupRecord $backup): bool
{
    return $backup->state === BackupState::Verified
        && $backup->bytes > 0
        && $backup->sourcePosition !== null
        && $backup->sourcePosition !== '';
}
~~~

This record is an inventory entry, not proof that a database engine can restore the bytes. `Verified` should be assigned only after integrity checks and a restore or equivalent validation appropriate to the backup type. Store the digest and source position separately from the secret used to access the backup.

## Retention and Recovery Windows

Retention is a policy over time, not a number of copies. Define:

* how many full and incremental or log segments are required;
* the oldest recovery point that must remain available;
* legal, contractual, and privacy deletion requirements;
* protection from accidental deletion or ransomware;
* how long encryption keys remain usable for retained copies;
* who can shorten retention or destroy a copy;
* how backup metadata maps a copy to a recovery procedure.

Keep more than one recovery point. A corruption or deletion may remain unnoticed until after the newest backup contains the damaged state. Retention must cover the detection window, investigation window, and restoration time, not only the time between scheduled jobs.

## Encryption and Independence

Encrypt backups in transit and at rest according to the data classification. Manage encryption keys as a separate dependency: a backup encrypted with an unavailable key is not usable. Keep key rotation and backup retention compatible, or re-encrypt retained copies under a controlled policy.

Independence has several dimensions:

* separate failure domain or account;
* separate credentials and authorization path;
* independent storage from the primary system;
* protection from ordinary application deletion;
* immutable or delayed-deletion retention where required;
* monitored access and audit evidence.

A backup mounted into the same host with the same root credentials is a copy, but it is weak protection against host compromise. A second region may share a control plane or identity failure. Understand the provider’s actual isolation guarantees.

## Restore Is the Test

Schedule restore validation, not only backup creation. A useful restore test:

1. select a declared recovery point without changing production;
2. obtain the required backup set, log segments, keys, and manifests;
3. restore into an isolated environment;
4. before write-path checks or queue replay, disable schedulers, outbound email, payments, webhooks, provider calls, and consumers, or reroute them to test doubles;
5. verify engine, schema, constraints, row or object counts, checksums, and known business invariants;
6. rebuild or attach derived stores according to the runbook;
7. replay or quarantine queue work under an explicit duplicate-effect policy;
8. exercise representative PHP read and write paths with outbound effects disabled;
9. measure elapsed time, operator work, resource consumption, and missing inputs;
10. record defects, owners, and the next test date.

A restore that is never tested is an assumption. A test that restores only a small table may prove tooling but not the production RTO.

## Database and PHP Boundaries

PHP should request or verify backups through a narrow administrative boundary, not implement engine recovery by copying files. A CLI command may create an application export, record a backup manifest, or run validation queries after an external restore tool completes.

Keep backup jobs separate from ordinary web requests. A request that waits for a multi-gigabyte export consumes PHP-FPM workers, request deadlines, memory, and database connections. Use a bounded job with progress, cancellation, retention, and access control. The job itself needs durable state so an interrupted run can be diagnosed and safely retried.

Do not expose backup archives through the public document root. Download access should require authorization, produce an audit record, and use bounded streaming. Treat exports as production data even when they are generated for support.

## Queues, Caches, and Derived State

Back up queue state only when its recovery semantics are defined. Replaying a message can duplicate an external effect. Preserve operation identity, acknowledgment state, lease information where meaningful, and the consumer contract. For some queues, retaining the source event or outbox is safer than copying broker internals.

Caches are often disposable, but cache keys can encode authorization or version assumptions. Rebuilding a cache after restore may create a load spike. Warm it gradually and protect the primary database with backpressure.

Search indexes and projections should have a rebuild procedure from authoritative data. Record the source position used for a rebuild so the result can be compared with the restored database. Do not promote a stale projection as if it were authoritative.

## Observability

Monitor the backup system with bounded, actionable signals:

* last successful capture per data class;
* age of the newest usable recovery point;
* log or change-stream continuity;
* bytes, duration, throughput, and failure category;
* encryption and key-access failures;
* retention and deletion events;
* restore duration and validation result;
* unresolved restore defects and missing dependencies.

Alert on recovery-point age and chain gaps, not only on a failed scheduler process. A job can exit successfully after backing up the wrong dataset or an empty directory. Compare the manifest to expected source identity, size range, schema version, and business sample.

## Performance and Capacity

Backups consume database I/O, CPU, storage, network bandwidth, locks, and provider quotas. Schedule them with workload awareness. A logical export that slows order writes may violate the service objective while reporting success.

Estimate capture and restore throughput:

~~~text
restore duration ≈ data to restore / sustainable restore throughput
             + log replay time
             + validation time
             + dependency and operator setup
~~~

Measure with production-shaped size and indexes. Compression trades CPU for storage and network cost. Deduplication can reduce bytes but adds dependency and recovery complexity. Preserve headroom for normal traffic during backup and for the recovery system’s load during restore.

## Security

Backups are high-value copies of production data. Apply least privilege to capture, read, restore, delete, and key-management operations. Separate the application’s runtime identity from the backup administrator and restore operator.

Protect manifests and logs from leaking database URLs, key material, personal data, or archive paths. Audit reads and restores. Test that a support user cannot browse arbitrary backup objects by changing an identifier.

Treat restored data as sensitive in the recovery environment. Mask or isolate personal and financial data when a full-fidelity copy is not required. Never use a production backup in a developer environment merely because it is convenient.

## Concurrency and Failure

Prevent overlapping jobs from corrupting a backup set or deleting a source still needed by another job. Use a durable job identity, lease, source-position check, and post-action verification. A retry after an ambiguous upload should verify the object digest before creating another copy.

Failure cases include:

* backup job runs while a schema migration changes the source;
* a log segment is deleted before the dependent base backup expires;
* storage reports success but the object is incomplete or unreadable;
* encryption-key rotation makes old copies inaccessible;
* retention cleanup races with a restore;
* restore succeeds but object metadata or configuration is missing;
* queue replay repeats a provider call;
* parallel restore tests exhaust the recovery environment;
* a backup account is compromised with permission to delete every copy.

Pause deletion when dependency or restore state is unknown. Preserve the failed set and evidence for investigation. A backup system should fail closed for destructive cleanup and fail visibly for capture or verification uncertainty.

## Testing Backups

Test more than the scheduler:

* verify expected datasets, source positions, schema versions, and non-empty content;
* corrupt, truncate, or remove a copy and confirm detection;
* restore full, incremental, and log-backed chains where used;
* restore after an interrupted migration and validate compatibility;
* test key rotation and retained-copy decryption;
* exercise retention deletion with legal and operational holds;
* simulate provider outage, quota exhaustion, and partial upload;
* restore object data and metadata together;
* replay or quarantine queue work with duplicate-effect protection;
* rebuild derived indexes and compare their source position;
* measure restoration at representative production size;
* verify audit records and least-privilege failures;
* repeat after database, backup-tool, runtime, and infrastructure upgrades.

Record the actual recovery point and elapsed restore time. Do not report an RPO or RTO from a design document after the implementation has changed without remeasuring it.

## Common Mistakes

* Equating a successful backup job with a tested restore.
* Keeping every copy in the same failure domain and account.
* Forgetting encryption keys, log segments, object metadata, or configuration.
* Copying a live database directory with PHP filesystem functions.
* Backing up derived state while omitting the authoritative source.
* Replaying queues without an idempotency and external-effect policy.
* Letting retention cleanup delete the only usable recovery chain.
* Testing restore with a tiny dataset and claiming the production RTO.
* Exposing archives through the web root or support interface.
* Running backups without capacity limits or workload coordination.
* Treating a stale or incomplete backup as a valid recovery point.
* Restoring production data into an uncontrolled development environment.

## Senior Engineer Thinking

The senior question is not “where is the backup file?” It is “what exact state can we recover, to which point, with which dependencies and keys, in how much time, under whose authority, and with what validation?”

Recoverability is an end-to-end property. It requires complete capture, independent protection, usable keys, retained history, documented dependencies, tested restoration, safe replay, capacity, and people who can execute the runbook. The restore test is the evidence that turns a backup from hope into an operational capability.

## Exercises

1. Inventory a PHP service’s database, uploads, queue, cache, search index, configuration, secrets, and deployment artifacts. Classify each as authoritative, derived, reproducible, or disposable.
2. Define RPO and RTO separately for login, order placement, reporting, and file download. Choose a backup and restore test for each.
3. Design a restore test after an accidental deletion that preserves writes after the recovery point for reconciliation.
4. Extend BackupRecord with retention expiry, encryption-key ID, schema version, and validation evidence. Define when cleanup must pause.

## Review Questions

* What makes a backup different from a recovery capability?
* How do RPO and RTO constrain backup and restore design?
* Why can a database backup be insufficient for a PHP application?
* What consistency boundary does a backup promise?
* When are logical, physical, change-log, and application-level backups useful?
* Why must encryption keys and log segments be retained with the backup policy?
* Why is restore validation more important than scheduler success?
* What risks arise when queue state is replayed after a restore?
* How should retention account for delayed corruption discovery?
* Which evidence proves that a claimed recovery point is usable?

## Summary

Backups are retained recovery inputs, not proof of recoverability. Inventory authoritative and derived state, define RPO and RTO per capability, choose consistent capture boundaries, protect copies and keys independently, retain complete chains, and test restoration at production-shaped scale. Include databases, logs, objects, configuration, secrets, deployment evidence, and queue semantics in the recovery design. Monitor recovery-point age and restore validation, control capacity and access, and record every unknown. A backup strategy is credible only when a tested restore can produce a validated state within the required loss and time boundaries.

## References

- [PostgreSQL 18: Backup and Restore](https://www.postgresql.org/docs/18/backup.html)
- [PostgreSQL: Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [MySQL 8.4: Backup and Recovery Types](https://dev.mysql.com/doc/refman/8.4/en/backup-types.html)
- [MySQL 8.4: Point-in-Time Recovery](https://dev.mysql.com/doc/refman/8.4/en/point-in-time-recovery.html)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 159 — Unit Tests](../11-testing/159-unit-tests.md)
- [Chapter 176 — Database Testing](../11-testing/176-database-testing.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 264 — Deployment](./264-deployment.md)
- [Chapter 265 — Rollback](./265-rollback.md)
- [Chapter 266 — CI/CD](./266-ci-cd.md)
- [Chapter 268 — Disaster Recovery](./268-disaster-recovery.md)
