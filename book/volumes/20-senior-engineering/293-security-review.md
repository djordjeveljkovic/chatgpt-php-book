---
book: The Complete Modern PHP Engineering Book
volume: 20
volume_title: SENIOR ENGINEERING
chapter: 293
title: Security Review
slug: security-review
status: complete
summary: ../../_ai/chapter-summaries/293-security-review-summary.md
---

# Chapter 293 — Security Review

## Why This Matters

A security review asks how an attacker could cross a trust boundary, alter an important decision, access protected data, or create an unsafe side effect—and whether the system provides effective prevention, detection, and recovery. It is not a checklist of dangerous functions.

Code review asks whether a concrete change is safe. Security review evaluates attacker capabilities, protected assets, abuse paths, controls, evidence, and residual risk. A feature can pass its happy-path tests while allowing cross-tenant access, replaying a webhook, leaking a secret, or exhausting a shared dependency.

## Define the Security Scope

Start with a security brief:

~~~text
capability:
assets:
actors:
trust boundaries:
attacker assumptions:
privileged operations:
external dependencies:
data sensitivity:
abuse scenarios:
deployment and rollback scope:
evidence and open questions:
~~~

Include affected tenants, administrators, background jobs, webhooks, exports, migrations, repair tools, caches, queues, and deployment configuration. A security review covers the paths an operator or worker can execute, not only the public HTTP controller.

## Threat Actors and Assets

Consider unauthenticated clients, malicious authenticated users, compromised accounts, hostile tenants, malicious uploaders and webhook senders, compromised dependencies, insiders, and compromised infrastructure. Assets include credentials, sessions, personal data, tenant data, payment records, queues, files, audit logs, signing keys, and database integrity.

For each actor ask what they can control, what they can observe, which rate or resource limit they can consume, and what durable or external effect they can request. Do not assume that authentication makes a user trustworthy; authorization and input constraints still apply.

## Authentication and Session Security

Review password hashing and verification, MFA and recovery, session fixation and rotation, expiration and revocation, CSRF, bearer-token storage, reset and verification tokens, login throttling, and account-enumeration behavior. Cookie settings such as `Secure`, `HttpOnly`, and an appropriate `SameSite` policy are part of the browser boundary.

Check that FPM, CLI, tests, and workers use compatible session and secret configuration. A security fix that works in the web SAPI but leaves an administrative CLI path unprotected is incomplete.

## Authorization and Object-Level Access

Make authorization explicit at the object and action boundary:

1. authenticate the actor;
2. derive tenant and resource scope from trusted server state;
3. authorize the specific operation;
4. load only records within that scope;
5. recheck sensitive transitions and external effects.

Hiding a button or checking a role in a controller is not sufficient. Every repository method, cache key, export, job, migration, and repair command must preserve scope.

This line deserves a security question:

~~~php
<?php

$order = $orders->find($request->integer('order_id'));
~~~

Can `find()` return another tenant’s order before authorization is applied? A scoped query such as `findForTenant($tenantId, $orderId)` makes the boundary harder to misuse, but it still needs a permission test and a review of every caller.

## Input Validation and Injection

Review separate defenses for SQL injection, shell commands, HTML and template output, headers, paths, LDAP or expression interpreters, redirects, logs, and regular expressions. Parameterization protects SQL values; it does not allow arbitrary identifiers or enforce authorization. Escaping is sink-specific and does not replace validation, bounds, or access control.

Validate type, size, cardinality, encoding, and allowed values at the boundary. Reject malformed input explicitly and avoid turning validation failure into a permissive fallback.

## SSRF and Outbound Requests

User-controlled URLs require more than checking the initial string. Review schemes, redirects, DNS rebinding, private and metadata addresses, response size, timeouts, egress controls, credential forwarding, and auditability. Validate the destination actually reached, not only the first address resolved.

An allowlist is useful only when its ownership and update process are clear. A request that can reach an internal service with ambient credentials is a trust-boundary change even when its response is never returned to the user.

## Uploads, Deserialization, and Dynamic Execution

For uploads, review size and count limits, MIME sniffing versus client extensions, random storage names, storage outside executable web roots, path normalization, archive extraction, parser isolation, tenant ownership, download authorization, and cleanup. Include zip bombs, polyglot files, symlinks, and unsafe filenames.

Treat `unserialize()` on untrusted input as a high-risk boundary. Signed data is not automatically safe if the signed producer is compromised or the accepted object graph is unsafe. Prefer explicit arrays or validated DTO-like structures for cookies, cache values, queues, and webhooks. Review dynamic class loading, `eval`, variable functions, and shell execution as separate controls.

## Secrets and Sensitive Data

Inspect secret-manager or environment delivery, repository and image leakage, rotation and revocation, least-privilege credentials, logs, exceptions, traces, queue payloads, backups, local copies, encryption boundaries, and access auditing. Ask who can read, rotate, revoke, and recover each secret. A debugging change that logs a whole request can turn a small defect into a lasting data exposure.

## Queues, Webhooks, and External Effects

Review webhook signature verification, timestamp and replay protection, idempotency keys, duplicate delivery, administrative-job authorization, queue confidentiality, poison messages, retry amplification, dead-letter access, provider response validation, and outbound credential scope.

Authenticating a webhook sender does not authorize every business operation in its payload. Verify that the event is intended for this tenant, accepted in its freshness window, and applied idempotently. For payments, notifications, and reservations, a timeout is an unknown outcome; reconciliation is part of the security and correctness boundary.

## Timing, Rate Limits, and Abuse Resistance

Review brute-force protection, per-account/IP/tenant/global limits, expensive search and export work, pagination and payload bounds, concurrency, queue admission, reset-token enumeration, timing-sensitive comparisons, and resource exhaustion through uploads, archives, regexes, and provider fan-out.

Rate limits must account for spoofable identity, shared NATs, distributed attackers, and storage outages. A limit that fails open may be an availability choice, but it must be explicit, bounded, observable, and safe for the protected action.

## Database, Cache, and Tenant Isolation

Check tenant predicates in every read and write, authorization-aware cache keys, background-job scope, exports, reports, search projections, repair scripts, database credentials, replicas, backups, and audit records for privileged changes. A missing tenant filter or cross-tenant cache collision is a data-exposure risk even if valid callers normally provide the right IDs.

The source of truth, cache, projection, and repair path must agree about identity. Security review follows data through the whole lifecycle rather than stopping at the controller.

## Supply Chain and Runtime Security

Review Composer lock integrity, abandoned or unsupported packages, transitive dependencies, PHP and extension versions, autoloading, build provenance, artifact integrity, vulnerability response, production debug settings, filesystem permissions, and FPM/CLI configuration drift. The newest dependency is not automatically safer; an update needs compatibility, test, deployment, and rollback evidence.

## Security Testing and Evidence

Match evidence to claims:

| Claim | Useful evidence |
| --- | --- |
| authorization | permission matrix and object-level integration tests |
| input safety | boundary, parser, fuzz, and injection tests |
| tenant isolation | cross-tenant query, cache, job, export, and repair tests |
| webhook/provider safety | signature, replay, contract, and idempotency tests |
| dependency safety | lock review, advisory scan, provenance, and runtime check |
| operational control | audit-log, alert, rollout, revocation, and recovery rehearsal |

Static analysis and line coverage are useful evidence, not proof of security. A passing happy path does not establish isolation, abuse resistance, or safe recovery.

## Findings and Risk Decisions

Make findings actionable:

~~~text
finding:
attacker capability:
affected asset:
exploit path:
evidence:
impact and likelihood:
scope:
severity:
remediation:
owner:
verification:
rollout and rollback:
accepted-risk expiry:
~~~

Separate blocker, high, medium, low, clarification question, and accepted risk. A question is not a severity level. Explain the unsafe behavior, consequence, and concrete correction; do not label every preference a vulnerability.

## Rollout and Incident Readiness

Security changes may alter sessions, authorization, tokens, schemas, caches, queues, or provider contracts. Review gradual rollout, old-worker compatibility, token and secret rotation, revocation, feature flags, audit and alert signals, rollback limits, forward recovery, incident escalation, evidence preservation, and customer or regulatory notification paths.

The rollback of a security fix can re-open exposure. The decision needs an explicit risk owner and a safer containment or forward-recovery path when binary reversal is unsafe.

## PHP Case Study

Suppose a tenant-scoped catalog application accepts an object ID, fills a cache, exports data asynchronously, processes uploads in a worker, and receives provider webhooks. The review discovers a cache key without tenant scope, an export job with overly broad credentials, a webhook retry without idempotency, and a legacy worker accepting an unvalidated serialized payload.

Identify each trust boundary, actor, asset, abuse path, missing control, test, rollout risk, and residual decision. The corrected design scopes the query and cache, narrows the export capability, authenticates and authorizes webhook operations, validates explicit message data, and records replay and audit evidence. It also verifies that old workers and rotated secrets can coexist during rollout.

## Common Mistakes

* checking authentication but not object-level authorization;
* trusting client-provided tenant IDs;
* treating hidden UI controls as access control;
* validating URLs without redirects, DNS rebinding, or egress analysis;
* accepting uploaded filenames or MIME types as trustworthy;
* using encoding as a substitute for validation and authorization;
* retrying webhooks or payments without idempotency;
* logging secrets or sensitive payloads during debugging;
* scanning dependencies without an upgrade and recovery plan;
* treating coverage as a security guarantee;
* ignoring workers, exports, migrations, repair scripts, and configuration;
* applying rate limits only at the IP layer;
* marking every question as a critical vulnerability;
* rolling back a security fix without understanding renewed exposure.

## Senior Engineer Thinking

Security review is threat modeling made operational. Follow actors and data across boundaries, ask what an attacker can control and observe, test prevention and detection at the real boundary, and make residual risk explicit. A secure-looking controller cannot compensate for a shared cache, privileged repair job, unsafe queue payload, or leaked secret.

## Exercises

1. Build a threat model for Chapter 287’s search service.
2. Create an authorization matrix for tenants, users, administrators, and support staff.
3. Review a user-configurable callback URL for SSRF and redirect risks.
4. Design an upload pipeline with processing and download boundaries.
5. Write webhook signature, freshness, replay, and idempotency checks.
6. Create permission-matrix tests for an export and repair command.
7. Review Composer, PHP, extension, and artifact supply-chain evidence.
8. Write a security finding with evidence, owner, severity, and verification.
9. Plan signing-key rotation or tightened authorization across mixed workers.
10. Create an incident checklist for suspected cross-tenant exposure.

## Review Questions

* How does security review differ from ordinary code review?
* Which actors and assets belong in a security brief?
* Why must authorization constrain the query and cache boundary?
* What makes an outbound URL review different from string validation?
* Why are uploads, deserialization, queues, and webhooks trust boundaries?
* How do rate limits protect both security and shared capacity?
* What evidence establishes tenant isolation?
* Why are coverage and dependency scans not security proof?
* How should a reviewer classify a blocker, a question, and accepted risk?
* What must be verified during rollout and after a security incident?

## Summary

Security review examines attackers, assets, trust boundaries, abuse paths, controls, evidence, rollout, and recovery. Review authentication and object-level authorization, follow tenant scope through data and effects, validate inputs for their sinks, protect outbound requests and uploads, avoid unsafe deserialization, secure secrets and dependencies, bound abuse, test real boundaries, make findings actionable, and record residual risk with owners and expiry.

## References

- [Chapter 142 — Security Model](../../volumes/10-security/142-security-model.md)
- [Chapter 147 — SSRF](../../volumes/10-security/147-ssrf.md)
- [Chapter 150 — File Upload Security](../../volumes/10-security/150-file-upload-security.md)
- [Chapter 152 — Password Security](../../volumes/10-security/152-password-security.md)
- [Chapter 155 — Secrets](../../volumes/10-security/155-secrets.md)
- [Chapter 157 — Supply-Chain Security](../../volumes/10-security/157-supply-chain-security.md)
- [Chapter 261 — Metrics](../../volumes/17-production-engineering/261-metrics.md)
- [Chapter 269 — Incident Response](../../volumes/17-production-engineering/269-incident-response.md)
- [Chapter 287 — Search/Filtering Service](../../volumes/19-small-engineering-projects/287-search-filtering-service.md)
- [Chapter 290 — Technical Debt](290-technical-debt.md)
- [Chapter 291 — Debugging Production](291-debugging-production.md)

## Chapter 294 Handoff

A security review identifies how harm could occur and which controls should prevent or detect it. When suspicious behavior or an actual failure appears in production, Chapter 294 will investigate the incident, preserve evidence, contain exposure, and coordinate recovery.
