---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 269
title: Incident Response
slug: incident-response
status: complete
summary: ../../_ai/chapter-summaries/269-incident-response-summary.md
---

# Chapter 269 — Incident Response

## Why This Matters

An incident is an unplanned event that threatens a service, customer, security, data, or compliance objective. Incident response is the coordinated work of detecting, understanding, containing, communicating, recovering, and learning from that event.

Technical skill alone does not make incident response effective. A team can have perfect dashboards and still lose time because nobody can declare the incident, choose priorities, authorize a risky change, communicate customer impact, or preserve evidence. Conversely, a team can restore traffic quickly while destroying forensic evidence or hiding an unresolved data-integrity problem.

Treat response as an operational system. Give it roles, authority, state, evidence, time boundaries, communication channels, and rehearsal. The goal is not to find someone to blame; it is to reduce harm, restore trustworthy service, and improve the system that made the incident possible.

NIST SP 800-61 Revision 3 is cybersecurity-focused guidance organized around Detect, Respond, and Recover, with continuous improvement across the broader risk-management lifecycle. This chapter applies the same discipline to reliability, data, security, and operational incidents without presenting one taxonomy or severity scale as universal.

## Mental Model

Response moves from signal to learning:

~~~text
signal or report
        ↓
validate and declare
        ↓
assign command and scope
        ↓
contain immediate harm
        ↓
diagnose while preserving evidence
        ↓
recover a bounded capability
        ↓
verify service and data integrity
        ↓
communicate closure and learn
~~~

Incident response is not the same as deployment, rollback, backup restore, or disaster recovery. Chapter 264 deploys, Chapter 265 chooses runtime recovery, Chapter 267 restores data, and Chapter 268 crosses a system failure boundary. Chapter 266 provides CI/CD evidence and promotion authorization. Chapter 269 coordinates people and decisions across those technical actions.

## Prepare Before the Incident

Preparation is response capability, not paperwork. Define:

* incident types and severity criteria;
* who may declare an incident and who may change severity;
* an incident commander and deputies;
* technical lead, communications lead, scribe, and subject-matter owners;
* escalation paths for security, legal, privacy, support, and suppliers;
* authoritative contact methods that do not depend on the failed service;
* service ownership, dependency maps, runbooks, and recovery targets;
* evidence retention and access policy;
* customer, partner, regulator, and internal notification requirements;
* handoff and closure rules.

Keep the plan discoverable during an outage. A runbook stored only in the unavailable production system is not an emergency control. Maintain a safe offline or independently reachable copy and test that responders can access it.

## Detect, Validate, and Declare

A signal can come from metrics, logs, traces, a health check, a customer, a provider, or an operator. Validate enough to avoid wasting scarce attention, but do not wait for perfect diagnosis before protecting customers.

Declare an incident when the event crosses the service, security, data, or operational threshold. Record:

* incident ID and declaration time;
* detector and initial symptom;
* affected service, capability, tenant, region, or data class;
* current customer and business impact;
* known safe and unsafe actions;
* initial severity and confidence;
* incident commander and active roles;
* next update time.

“Investigating” without an owner or next update is not a response state. If the signal is later downgraded, preserve the declaration and reasoning so alert quality can improve without discouraging reports.

## Severity and Priorities

Severity should express impact and urgency, not the seniority of the person who noticed the issue. Define a small policy, for example:

| Severity | Typical impact | Immediate priority |
| --- | --- | --- |
| Critical | broad outage, active compromise, or unsafe data effects | protect life, trust, security, and core invariants |
| High | major capability unavailable or rapidly worsening | contain blast radius and restore a bounded capability |
| Moderate | limited customers, degraded performance, or recoverable queue growth | assign owner and prevent escalation |
| Low | localized defect with workaround | schedule repair and monitor |

The labels are organization-specific. A security incident with no visible outage can be critical because evidence and containment are time-sensitive. A billing invariant violation can outrank a larger but reversible latency regression.

Prioritize by harm: safety and security, data integrity, core customer capability, durable work, then convenience and diagnostics. Do not optimize a dashboard while a business invariant is being violated.

## Roles and Decision Rights

Separate coordination from implementation:

* Incident commander owns priorities, decision cadence, escalation, and the declaration that response has ended.
* Technical lead coordinates diagnosis, containment, changes, and verification.
* Communications lead prepares internal, customer, partner, and regulator updates.
* Scribe maintains the timeline, decisions, evidence references, and owners.
* Service and dependency owners provide bounded technical actions and risk assessments.
* Security, privacy, legal, and support leads advise on specialized obligations.

The commander does not need to be the person with the deepest PHP knowledge. The role is to keep the response coherent and ensure that technical decisions have an owner and an exit condition.

Use explicit decision rights. The person proposing a production change should state its expected benefit, risk, scope, verification, rollback or containment path, and deadline. The commander or delegated authority decides whether it is authorized.

## A Typed Incident State

Use a small state model for coordination, not as a substitute for human judgment:

~~~php
<?php

declare(strict_types=1);

enum IncidentPhase: string
{
    case Reported = 'reported';
    case Declared = 'declared';
    case Containing = 'containing';
    case Recovering = 'recovering';
    case Monitoring = 'monitoring';
    case Closed = 'closed';
}

final readonly class IncidentRecord
{
    public function __construct(
        public string $incidentId,
        public IncidentPhase $phase,
        public string $commander,
        public string $summary,
        public int $affectedUsers,
        public bool $customerImpactKnown,
        public string $nextUpdateAt,
    ) {
        if ($incidentId === '' || $commander === '' || $summary === '') {
            throw new InvalidArgumentException('Incident identity is incomplete');
        }

        if ($affectedUsers < 0) {
            throw new InvalidArgumentException('Affected-user count cannot be negative');
        }

        if ($phase !== IncidentPhase::Closed && $nextUpdateAt === '') {
            throw new InvalidArgumentException('Active incident needs a next update');
        }
    }
}

function mayCloseIncident(
    IncidentRecord $incident,
    bool $serviceVerified,
    bool $ownersAssigned,
    bool $effectsReconciled,
    bool $communicationsComplete,
    bool $commanderAuthorized,
): bool
{
    return $incident->phase === IncidentPhase::Monitoring
        && $serviceVerified
        && $ownersAssigned
        && $effectsReconciled
        && $communicationsComplete
        && $commanderAuthorized
        && $incident->customerImpactKnown;
}
~~~

The record keeps coordination facts explicit: an active incident has an owner and a next update, while closure requires verified service and known impact. A real system should store an append-only decision history, role changes, evidence references, approvals, and unresolved follow-up work rather than overwriting one mutable row.

## Containment

Containment reduces harm before complete diagnosis. Examples include:

* pause a rollout or disable a feature flag;
* remove a failing dependency path or shed noncritical work;
* stop a queue producer or quarantine a poison message;
* revoke a compromised credential and preserve affected evidence;
* isolate a tenant, region, host, or network boundary;
* switch to read-only or degraded service;
* fence a stale writer before failover;
* rate-limit an abusive or expensive operation.

Every containment action has a cost. Disabling order placement protects inventory integrity but may increase customer harm through unavailability. Record the expected trade-off and a review time. A temporary bypass without an owner tends to become permanent production behavior.

## Diagnose Without Destroying Evidence

Diagnose in parallel with containment. Use bounded queries and preserve:

* deployment, configuration, feature-flag, and secret-key changes;
* release, artifact, schema, and migration identities;
* logs, metrics, traces, health states, and queue observations;
* request, operation, message, and correlation identifiers;
* access records, audit events, and provider responses;
* host, container, PHP-FPM, Nginx, and database state;
* exact commands, timestamps, actors, and resulting observations.

Do not restart, delete, rotate, or clean up automatically when the action may erase evidence or alter the failure. If security compromise is plausible, involve the security owner before collecting or modifying affected systems. Minimize access to sensitive evidence and record chain of custody where required.

Prefer hypotheses that make predictions: “the new release fails only when the expanded column is null” can be tested against release, schema, route, and outcome dimensions. Avoid declaring root cause from one correlated log line.

## Technical Recovery Coordination

Choose the technical path from the incident state:

* Chapter 264 deployment pause or repair for a rollout problem;
* Chapter 265 rollback, roll-forward, or reconciliation for a bad release;
* Chapter 267 restore for data loss or corruption;
* Chapter 268 failover or degraded operation for a failure-domain event;
* a provider escalation or customer-workaround for an external dependency;
* a security containment and clean rebuild for a compromise.

The incident commander coordinates the decision; the technical owner executes a bounded action. Require a pre-change statement:

~~~text
action: disable checkout writes
reason: duplicate inventory reservations detected
scope: checkout API in region-eu
expected benefit: stop new inconsistent reservations
risk: customers receive pending status
verification: reservation invariant and queue age
next decision: in 15 minutes
owner: service-team
~~~

After the action, record what actually happened. A command’s exit status is evidence about the command, not proof that traffic, data, or customers changed as expected.

## Communication

Communication is part of recovery. Give each audience useful truth at the right level:

* responders need current evidence, hypotheses, owners, and next actions;
* executives need impact, risk, decisions, and forecast uncertainty;
* support needs customer-visible symptoms and safe workarounds;
* customers need affected capability, start time, current mitigation, and next update;
* partners and regulators may need contractually or legally defined notices;
* the public status page should avoid sensitive details while remaining accurate.

Use a regular update cadence even when there is no major change. Say “we have not confirmed whether writes after 14:05 are complete” rather than converting uncertainty into reassurance. Do not publish personal data, credentials, exploit details, or unverified root-cause claims.

## Handoffs and Fatigue

Incidents outlast individuals. A handoff should state:

* current phase and severity;
* customer and business impact;
* actions completed and their observed results;
* active hypotheses and evidence links;
* unsafe or prohibited actions;
* pending decisions and deadlines;
* owners and next communication time.

Rotate responders before exhaustion causes unsafe changes. Keep the scribe and commander roles covered. A tired engineer should be able to read the record and understand what is safe without relying on a private chat or memory.

## Security and Privacy Incidents

Treat suspected compromise differently from an ordinary availability defect. Preserve evidence, restrict access, avoid tipping off an attacker unnecessarily, and coordinate with security, legal, privacy, and communications owners. Credential rotation, host isolation, log collection, and clean rebuilds can affect availability and forensic value; make the trade-offs explicit.

Do not use incident channels as an uncontrolled place to paste customer data or secrets. Redact tokens and personal data, limit membership, protect exported timelines, and apply retention rules. A post-incident report should explain impact without publishing an exploit recipe or sensitive identifiers.

## Capacity and Performance

Response consumes production and human capacity. Debug queries can increase database load; verbose logging can fill disks; repeated retries can amplify a provider outage; emergency deployments can compete with recovery traffic. Bound diagnostic work and use read-only or sampled evidence where possible.

Track response latency separately from service latency: time to detect, declare, assign, contain, mitigate, recover, verify, and communicate. A short time to “green” is not success if reconciliation or customer notification remains incomplete.

## Concurrency and Change Control

One incident can attract several well-intentioned operators. Use the incident record and change ownership to prevent conflicting actions. Require expected state, scope, and post-change verification for production changes. Do not allow a stale responder to undo a containment action after handoff without coordination.

Automated remediation needs the same boundaries as human action: rate limits, scope, authorization, idempotency, circuit breaking, and a stop condition. If automation cannot observe its effect, it should pause or escalate rather than repeat indefinitely.

## Closure and Learning

Closure is more than service recovery. Before closing:

* the agreed capability is verified;
* customer and business impact is measured as far as possible;
* durable and external effects are reconciled or explicitly owned;
* temporary flags, routes, credentials, and containment actions have owners;
* evidence and the timeline are retained safely;
* customer, partner, and internal updates are complete;
* follow-up actions have owners, priority, and due dates;
* a review date is scheduled.

Capture lessons during the response when an action, assumption, or missing signal becomes visible; do not rely on memory after the channel closes. The post-incident review should ask which conditions allowed the incident, which detection or control failed, which recovery action helped, and which proposed fix reduces future harm. Avoid “add more monitoring” without naming the decision the signal will support.

## Testing Incident Response

Exercise the human and technical system:

* run a tabletop for a PHP-FPM outage, queue poison message, data corruption, or credential compromise;
* declare incidents from synthetic alerts and customer reports;
* test role assignment, paging, offline runbook access, and handoffs;
* rehearse deployment pause, rollback, data restore, and regional failover;
* inject stale or missing telemetry and practice uncertainty statements;
* test communication templates and status-update cadence;
* verify that incident evidence does not expose secrets or personal data;
* force ambiguous production actions and require post-action verification;
* measure time to detect, declare, contain, restore, verify, and reconcile;
* review and retire stale runbooks, contacts, permissions, and assumptions.

Exercise one failure mode at a time before combining them. A complicated game day that nobody can diagnose produces less learning than a bounded drill with a clear success criterion.

## Common Mistakes

* Waiting for root cause before declaring or containing an incident.
* Having several people make production changes without one coordinator.
* Treating the loudest metric as the highest business impact.
* Losing evidence through automatic restarts, cleanup, or log rotation.
* Communicating only when there is good news.
* Publishing unverified root cause, personal data, or sensitive security details.
* Closing when traffic is restored but data and external effects are unreconciled.
* Keeping temporary emergency flags, credentials, or bypasses forever.
* Depending on a failed service for runbooks, paging, or communication.
* Running diagnostics without capacity limits during an outage.
* Treating a tabletop as proof that a technical failover works.
* Writing follow-up actions with no owner, due date, or decision rationale.

## Senior Engineer Thinking

The senior question is not “who caused the incident?” It is “what harm is occurring now, what authority and evidence do we need to reduce it, which actions are safe under uncertainty, and how will we know that recovery is trustworthy?”

Incident response turns partial information into coordinated action. Strong teams make uncertainty visible, keep changes bounded, protect evidence, communicate honestly, and learn without blame. Their systems are designed so that one person can declare an incident, another can execute a safe technical action, and the organization can verify and improve the result.

## Exercises

1. Write an incident record for a PHP-FPM deployment that causes elevated checkout errors and duplicate queue deliveries. Assign roles, severity, containment, evidence, updates, and closure criteria.
2. Design a handoff from a night responder to a daytime incident commander. Include current impact, hypotheses, unsafe actions, pending decisions, and evidence links.
3. Draft customer, support, executive, and internal updates for a data-integrity incident where the recovery point is known but post-cutoff writes are not reconciled.
4. Run a tabletop in which the monitoring system and primary communication channel are unavailable. Identify independent controls and escalation paths.

## Review Questions

* What makes incident response different from deployment, rollback, backup restore, and disaster recovery?
* When should a team declare an incident before it knows the root cause?
* Why must coordination and implementation roles be separate?
* What belongs in the incident record and timeline?
* How do containment actions trade availability against integrity or security?
* Why should diagnostics preserve evidence and have capacity limits?
* What makes a customer update honest under uncertainty?
* Why is service traffic restoration insufficient for closure?
* How should handoffs and responder fatigue affect safety?
* What evidence demonstrates that an incident-response exercise improved capability?

## Summary

Incident response is coordinated action under uncertainty. Prepare roles, authority, independent communication, runbooks, evidence policies, severity criteria, and recovery boundaries before an incident. Detect and declare early enough to contain harm, assign command, preserve evidence, make bounded changes, communicate impact and uncertainty, coordinate rollback/restore/failover, verify service and data integrity, reconcile effects, and close only with owned follow-up. Rehearsal turns a document into a capability, while blameless learning turns an incident into a stronger system.

## References

- [NIST SP 800-61 Rev. 3: Incident Response Recommendations](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [NIST Incident Response project](https://csrc.nist.gov/projects/incident-response)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 235 — Scaling](../15-performance/235-scaling.md)
- [Chapter 241 — Partial Failure](../16-distributed-systems/241-partial-failure.md)
- [Chapter 242 — Idempotency](../16-distributed-systems/242-idempotency.md)
- [Chapter 243 — Message Delivery](../16-distributed-systems/243-message-delivery.md)
- [Chapter 249 — Distributed Locks](../16-distributed-systems/249-distributed-locks.md)
- [Chapter 259 — Secrets](./259-secrets.md)
- [Chapter 260 — Logging](./260-logging.md)
- [Chapter 261 — Metrics](./261-metrics.md)
- [Chapter 262 — Tracing](./262-tracing.md)
- [Chapter 263 — Health Checks](./263-health-checks.md)
- [Chapter 264 — Deployment](./264-deployment.md)
- [Chapter 265 — Rollback](./265-rollback.md)
- [Chapter 267 — Backups](./267-backups.md)
- [Chapter 268 — Disaster Recovery](./268-disaster-recovery.md)
