---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 268
title: Disaster Recovery
slug: disaster-recovery
status: complete
summary: ../../_ai/chapter-summaries/268-disaster-recovery-summary.md
---

# Chapter 268 — Disaster Recovery

## Why This Matters

Disaster recovery is the capability to restore an acceptable service after a failure exceeds the normal operating boundary. The failure may be a lost host, region, account, network, identity provider, database cluster, artifact registry, or critical supplier. It may also be a security event in which continuing to use the original environment would spread compromise.

A database restore alone is not disaster recovery. The recovered system needs code, configuration, secrets, networking, identity, queues, observability, operators, and a safe way to serve customers. A warm replica can reduce database recovery time while the application’s deployment account, DNS, certificate, or payment provider remains unavailable.

Disaster recovery is a system property with explicit trade-offs. Define which capabilities must return, to what level, by when, with how much data loss, and under whose authority. Then exercise the plan against the failure domains it claims to cover.

## Mental Model

Recovery crosses several boundaries:

~~~text
failure detected
        ↓ classify scope and safety
contain affected domain
        ↓ choose continuity mode
fail over | restore | degrade | wait
        ↓ establish dependencies
identity, network, code, data, queues, observability
        ↓ verify invariants and capacity
serve a bounded capability
        ↓ reconcile and fail back deliberately
normal operations
~~~

Chapter 267 covers backup and restore inputs. Chapter 268 composes those inputs into a service recovery plan. Chapter 269 will cover human incident command and communication. Disaster recovery needs both technical automation and a human authority to decide when the recovered system is safe to expose.

## Define the Recovery Contract

For each capability, record:

* minimum service level during recovery;
* recovery time objective and recovery point objective;
* acceptable data loss and reconciliation policy;
* dependencies and their failure domains;
* failover trigger and authority;
* required capacity and warm-up time;
* security and access prerequisites;
* validation evidence before traffic restoration;
* customer and partner communication requirements;
* failback conditions and ownership.

“The website should be back” is not a contract. “Customers can view existing orders within two hours, while new order placement remains disabled until the ledger is reconciled” is closer to one.

RPO is a required maximum age for a usable recovery point, not proof of the actual point available during an incident. The actual data loss may be older than the objective or unknown. RTO is also a contract boundary: define whether its clock starts at detection or incident declaration and ends when the capability meets its agreed service level with validation evidence. It includes decision, provisioning, artifact retrieval, database restore or promotion, log replay, secret access, DNS or routing convergence, cache warm-up, smoke checks, operator coordination, and traffic ramp. Optimizing one step does not prove the total objective.

## Failure Domains

Map the system’s dependencies to failure domains:

| Domain | Example failure | Recovery question |
| --- | --- | --- |
| Process | PHP-FPM crash or bad release | Can normal deployment or rollback recover it? |
| Host | disk, kernel, or machine loss | Can another host serve the capability? |
| Zone | power or network isolation | Is capacity and state available elsewhere? |
| Region | routing, cloud, or facility outage | Can the service use an independent region? |
| Control plane | deployment, DNS, or identity outage | Can operators still change or observe state? |
| Data system | corruption, deletion, or replication failure | Can a validated recovery point be promoted? |
| Supplier | payment, email, DNS, or cloud dependency outage | Can the capability degrade or switch providers? |
| Security boundary | credential compromise or ransomware | Can the clean environment be trusted and accessed? |

Redundancy inside one failure domain is not independence from that domain. Two containers on one host do not cover host loss. Two databases replicating immediately may share corruption. Two regions using the same identity account may fail together.

## Recovery Modes

### Restore and rebuild

Provision a clean environment, retrieve artifacts and backups, restore authoritative data, rebuild derived state, configure dependencies, validate, and expose a bounded capability. This can cover large failures but has the longest and most operationally complex path.

### Warm standby

Maintain a partially or fully provisioned secondary environment and keep required data sufficiently current. This reduces provisioning time but costs capacity and introduces replication, drift, credential, and maintenance risks.

### Hot standby or active-passive

Keep a fully provisioned secondary ready to take traffic while the primary remains the write owner. This can reduce traffic-switch time, but it costs capacity and still requires fencing, promotion, and dependency validation.

### Active-active

Keep multiple populations serving traffic at the same time. This can reduce a single-site traffic switch, but split-brain, duplicate effects, data conflicts, session routing, and shared dependency failures remain possible. Active-active is not a free substitute for a recovery plan.

### Degraded operation

Serve a smaller capability: read-only orders, cached catalog pages, delayed reports, or local queue acceptance without immediate fulfillment. Degradation needs explicit authorization, customer semantics, data guarantees, and a path back to full service.

Choose the least complex mode that meets the capability contract. Complexity itself adds failure modes and testing cost.

## A Typed Recovery Decision

Keep the decision policy separate from provider APIs and failover mechanisms:

~~~php
<?php

declare(strict_types=1);

enum ContinuityMode: string
{
    case Normal = 'normal';
    case Degraded = 'degraded';
    case Failover = 'failover';
    case Restore = 'restore';
    case Halt = 'halt';
}

final readonly class RecoveryState
{
    public function __construct(
        public string $incidentId,
        public bool $primaryTrusted,
        public bool $secondaryReady,
        public bool $dataValidated,
        public bool $identityAvailable,
        public bool $telemetryFresh,
        public bool $fencingConfirmed,
        public bool $trafficControlAvailable,
        public bool $capacityReady,
        public bool $recoveryAuthorized,
    ) {
        if ($incidentId === '') {
            throw new InvalidArgumentException('Invalid recovery state');
        }
    }
}

function chooseContinuityMode(RecoveryState $state): ContinuityMode
{
    if (!$state->telemetryFresh) {
        return ContinuityMode::Halt;
    }

    if ($state->primaryTrusted) {
        return ContinuityMode::Normal;
    }

    if (
        $state->secondaryReady
        && $state->dataValidated
        && $state->identityAvailable
        && $state->fencingConfirmed
        && $state->trafficControlAvailable
        && $state->capacityReady
        && $state->recoveryAuthorized
    ) {
        return ContinuityMode::Failover;
    }

    if (
        $state->dataValidated
        && $state->identityAvailable
        && $state->trafficControlAvailable
        && $state->capacityReady
        && $state->recoveryAuthorized
    ) {
        return ContinuityMode::Degraded;
    }

    return ContinuityMode::Restore;
}
~~~

This is a deliberately small policy. It does not decide whether a specific region, database timeline, credential, or provider is safe. The real contract needs dependency freshness, capacity, traffic scope, split-brain protection, reconciliation state, authorization, and an explicit manual-intervention path.

## Failover Sequence

Failover is a state transition, not a single DNS change:

1. declare the affected domain and recovery owner;
2. freeze unsafe writes, deployments, schedulers, and automation;
3. determine whether the primary is still trusted and whether it must be fenced;
4. select the recovery point or replica timeline;
5. establish identity, networking, secrets, artifacts, and observability;
6. promote or restore data under the declared consistency policy;
7. verify schema, business invariants, queues, and external-effect boundaries;
8. establish capacity and disable unsafe outbound effects where necessary;
9. shift a bounded traffic or tenant cohort;
10. observe service and business signals before expanding;
11. record unreconciled work and announce capability limits;
12. plan failback only after the recovered system is stable.

Do not allow both primary and secondary to accept conflicting writes unless the data model and conflict policy explicitly support it. If fencing cannot be proved, prefer read-only or halt modes over split-brain writes.

## Data and Replication

Replication can reduce data loss and recovery time, but it also copies some failures. A deletion, corruption, bad migration, or malicious write may replicate immediately. Maintain independent recovery points and know the replication lag, timeline, conflict policy, and promotion procedure.

After a data promotion, capture the database- and provider-specific source position and timeline. New writes may diverge from the former primary. Do not simply reattach the old primary without comparing or discarding divergent state according to a reviewed procedure.

Object storage, databases, queues, and search projections can have different recovery points. Define whether the recovered service is read-only until those points are reconciled. Preserve operation identity, durable idempotency records, and reconciliation evidence; an operation ID alone does not prevent a duplicate payment, email, inventory change, or webhook.

## Dependencies and Service Levels

List dependencies by capability and recovery mode:

| Capability | Required dependencies | Degraded option |
| --- | --- | --- |
| View orders | database, identity, PHP application | cached or read-only view |
| Place order | database, payment provider, inventory, queue | disable or accept pending work |
| Send email | queue, provider credentials, template artifact | defer and replay safely |
| Admin repair | identity, audit store, database, observability | manual break-glass procedure |

An application can be healthy while a required provider is unavailable. A recovered region that cannot authenticate users or publish durable events is not equivalent to normal service. Declare the capability boundary to customers and operators.

## PHP Runtime and Infrastructure

The recovered environment needs the same or compatible PHP runtime, extensions, application artifact, configuration schema, secret references, FPM capacity, Nginx or ingress behavior, and writable runtime directories. Chapter 264’s deployment checks and Chapter 267’s restore validation must run in the recovery environment, not only in the primary.

Do not assume a container image contains everything required for recovery. Registries, base-image mirrors, Composer caches, certificate authorities, DNS, and identity providers may be part of the failed domain. Retain approved artifacts in an independently reachable location and test access from the recovery boundary.

Disable scheduled jobs and duplicate consumers until ownership is clear. A recovered environment starting every scheduler and worker at once can double external effects or overwhelm the restored database.

## Failback

Failback returns service to the preferred environment after the original failure is understood and the target is safe. It is another migration with compatibility, data, traffic, and capacity risks.

Before failback:

* identify all writes accepted by the recovery environment;
* reconcile or deliberately discard divergent state under policy;
* validate the former primary’s integrity and security;
* make the target schema and artifacts compatible;
* stop or fence competing writers;
* rehearse the traffic and queue transition;
* retain recovery evidence and a new rollback target.

Do not fail back merely because the preferred region is reachable. Reachability is not integrity, capacity, or readiness.

## Observability and Communication

Recovery needs evidence across the entire path:

* detection time and affected failure domain;
* recovery owner and authorization;
* current primary/secondary role and fencing state;
* backup or replica source position and validation result;
* dependency availability and capability mode;
* traffic scope, queue state, and data divergence;
* error, latency, saturation, and business invariants;
* customer impact, reconciliation backlog, and unresolved unknowns;
* failover and failback timestamps.

Keep recovery events durable and correlated with logs, metrics, traces, deployment records, and audit records. Chapter 269 covers the human communication plan; the technical system should still emit the facts needed for that plan.

## Performance and Capacity

Disaster recovery often needs more temporary capacity than normal operation. Restore and replay consume I/O, CPU, memory, database connections, queue throughput, network bandwidth, and operator attention. A secondary that can serve normal traffic may not have enough capacity for restore validation, backfill, replay, and traffic simultaneously.

Measure the full RTO path. Provisioning time, artifact fetch, restore throughput, log replay, index rebuild, cache warm-up, DNS convergence, certificate issuance, and validation each need a budget. Add margin for retries and degraded dependencies.

Load the recovered system gradually. A cold cache, newly promoted database, or newly started PHP-FPM fleet can make the first traffic wave much more expensive than steady state. Protect the recovery environment with admission limits and backpressure.

## Security

Disaster recovery is also a security boundary. A recovery account with unrestricted access to every backup, secret, region, and deployment control is a high-impact credential. Separate duties for provisioning, data restore, secret access, traffic change, and approval where practical.

If the incident may be a compromise, do not restore unverified artifacts, credentials, workflow state, or infrastructure images. Establish a clean trust root, rotate affected credentials, preserve forensic evidence, and verify the source and artifact chain before serving.

Break-glass access should be short-lived, logged, scoped, and reviewed. Protect recovery environments from becoming permanent copies of production data with weaker access controls. Apply the privacy and retention policy to restored data and temporary replicas.

## Concurrency and Control

Use one recovery controller or an explicit coordination protocol for each service and environment. Fence stale owners before enabling writes in the secondary. A lease without fencing may prevent two well-behaved controllers from acting, but it cannot stop a disconnected primary from accepting writes unless the data or network boundary enforces it.

Make failover and failback operations idempotent. “Set the active region to secondary at expected epoch 12” is safer than “toggle to the other region.” Persist the epoch, source position, traffic scope, capability mode, and validation evidence. After every ambiguous action, query observed state before retrying.

## Testing and Exercises

Test disaster recovery as a system:

* lose one PHP host and verify ordinary deployment recovery;
* isolate a zone and measure capacity and routing behavior;
* make the primary database unavailable and promote a validated secondary;
* restore from backups when the primary region and registry are inaccessible;
* simulate identity, DNS, certificate, object-storage, queue, and provider failures;
* fence the former primary and test split-brain prevention;
* replay or reconcile writes after a failover;
* start the recovered PHP-FPM and worker populations with schedulers disabled;
* measure the full RTO and actual data-loss window;
* exercise degraded read-only and pending-work modes;
* fail back after divergent writes and verify the reconciliation procedure;
* run the plan after runtime, schema, backup, identity, and infrastructure changes.

A tabletop is useful, but it does not replace a controlled technical failover. A technical failover is useful, but it does not replace an authorized decision and communication plan.

## Common Mistakes

* Calling a second replica a disaster-recovery plan without testing promotion.
* Assuming replication protects against corruption or malicious writes.
* Measuring only database restore time instead of end-to-end RTO.
* Forgetting DNS, identity, certificates, artifacts, secrets, queues, or observability.
* Allowing primary and secondary to accept conflicting writes.
* Starting schedulers and workers before ownership is clear.
* Serving full traffic before capacity and invariants are validated.
* Failing back as soon as the old region becomes reachable.
* Restoring compromised artifacts or credentials into the clean environment.
* Treating degraded service as normal service without documenting its guarantees.
* Running a tabletop without measuring a real restore or failover.
* Leaving temporary recovery copies with permanent production access.

## Senior Engineer Thinking

The senior question is not “where is our backup region?” It is “which failure domain are we escaping, what state and authority move with us, which capabilities return first, how do we fence the old writers, and what evidence proves the recovered system is safe?”

Disaster recovery is a composition of data recovery, infrastructure provisioning, identity, networking, application deployment, traffic control, dependency behavior, security, capacity, and human authority. The plan is credible only when those boundaries have been exercised together and the remaining unknowns are explicit.

## Exercises

1. Define recovery contracts for order viewing, order placement, reporting, and email delivery. Assign RPO, RTO, degraded behavior, dependencies, and validation evidence.
2. Draw the state transition for a regional failover with a database replica, queue, object storage, DNS, and PHP-FPM service. Mark the fencing points.
3. Calculate an end-to-end RTO budget from provisioning, artifact retrieval, restore, replay, warm-up, validation, and traffic ramp.
4. Design a failback drill with divergent writes. Specify the reconciliation owner, accepted data loss, and stop conditions.

## Review Questions

* How is disaster recovery different from a normal rollback or backup restore?
* Why is RTO an end-to-end property?
* What makes a failure domain independent?
* Why can replication copy corruption or deletion?
* Which dependencies must be available before a PHP service can fail over?
* Why must the former primary be fenced before enabling writes elsewhere?
* What is the purpose of degraded service modes?
* Why is failback another migration rather than a simple switch?
* Which evidence proves a recovered system is safe to expose?
* Why do technical failover drills and tabletop exercises complement each other?

## Summary

Disaster recovery composes backups, infrastructure, identity, networking, code, secrets, data, queues, dependencies, traffic, capacity, security, observability, and human authority into a system-wide recovery capability. Define per-capability RPO/RTO and degraded contracts, map failure domains, choose a recovery mode, fence conflicting writers, restore or promote validated state, ramp traffic gradually, reconcile divergence, and fail back deliberately. A secondary region or replica is only an input; recoverability is proven by an exercised end-to-end transition with explicit evidence and owned unknowns.

## References

- [NIST SP 800-34 Rev. 1: Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final)
- [PostgreSQL: Backup and Restore](https://www.postgresql.org/docs/18/backup.html)
- [PostgreSQL: Continuous Archiving and Point-in-Time Recovery](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [PostgreSQL: High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL 8.4: Backup and Recovery Types](https://dev.mysql.com/doc/refman/8.4/en/backup-types.html)
- [MySQL 8.4: Replication](https://dev.mysql.com/doc/refman/8.4/en/replication.html)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 264 — Deployment](./264-deployment.md)
- [Chapter 265 — Rollback](./265-rollback.md)
- [Chapter 266 — CI/CD](./266-ci-cd.md)
- [Chapter 267 — Backups](./267-backups.md)
- [Chapter 269 — Incident Response](./269-incident-response.md)
