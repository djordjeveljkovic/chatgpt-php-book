---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 142
title: Security Model
slug: security-model
status: complete
summary: ../../_ai/chapter-summaries/142-security-model-summary.md
---

# Chapter 142 — Security Model

## Why This Matters

Application security is a property of a system, not a middleware switch. A PHP endpoint may be syntactically correct and still expose another tenant's data, accept a forged state transition, leak credentials in logs, or fail open when a dependency is unavailable.

A security model gives the team a shared way to identify assets, trust boundaries, actors, threats, controls, and evidence. It turns “make it secure” into decisions that can be implemented and tested. The model must follow data through HTTP, PHP, databases, queues, caches, files, and operators.

## Assets, Actors, and Trust Boundaries

Start with what needs protection: credentials, personal data, money, tenant records, availability, integrity of configuration, and audit evidence. Identify actors and their capabilities: anonymous visitors, authenticated users, administrators, support staff, service accounts, third-party providers, and attackers. Then draw trust boundaries where assumptions change.

```text
browser ──TLS──> web server ──process boundary──> PHP application
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                      database      queue       object store
```

A request header is not trusted merely because it arrived at the application. A queue payload is not trusted merely because an internal worker produced it; workers are deployed independently and messages may be old or malformed. A database row is authoritative for some facts, but user-controlled text stored in it remains untrusted when rendered later.

For each boundary, state the guarantee you rely on. TLS protects transport in a configured path; it does not authorize a user. A private network limits exposure; it does not make a service safe from compromised credentials. A database constraint protects a defined invariant; it does not validate an HTML response.

## Threat Modeling

Use a lightweight threat model before implementing a high-impact flow. List assets, entry points, trust boundaries, abuse cases, and mitigations. STRIDE is one useful vocabulary: spoofing, tampering, repudiation, information disclosure, denial of service, and elevation of privilege. The label matters less than considering each failure mode and assigning an owner.

A payment endpoint, for example, has threats beyond SQL injection: an attacker can replay a request, change the account identifier, enumerate orders, exhaust provider quota, or exploit a webhook that is trusted without signature verification. Map controls to the concrete threat and preserve evidence that the control is active.

Risk is a function of impact, likelihood, exposure, and detectability. Do not treat a low-probability threat as irrelevant when its impact is account takeover or irreversible data loss. Conversely, controls also have availability and operational costs. Record the decision and revisit it when the threat, dependency, or data changes.

## Security Properties and Control Layers

The confidentiality, integrity, and availability model is a useful starting point:

* **Confidentiality:** only authorized subjects and processes can read data.
* **Integrity:** unauthorized or invalid changes are rejected and detected.
* **Availability:** legitimate work can complete within an acceptable service level.

Add authenticity and accountability where the domain needs them. Use defense in depth with independent layers: authentication, authorization, input validation, output encoding, database constraints, least-privilege credentials, rate limits, timeouts, audit events, monitoring, and recovery. A second layer should reduce impact when the first fails; copying the same unchecked claim into two controllers is not defense in depth.

Secure defaults matter. Deny access without a valid decision, reject malformed input, use restrictive cookie flags, avoid verbose production errors, set finite timeouts, and require explicit opt-in for dangerous operations. Make exceptional access visible and expiring rather than silently broadening the normal path.

## Identity, Authorization, and Data Flow

Authentication establishes an identity. Authorization decides whether that identity may perform an action on a particular resource in a context. Keep the decision tied to the actual object and tenant boundary; never accept `user_id`, `role`, or `is_admin` from a request as proof.

Classify data by sensitivity and define where it may flow. A password reset token can be accepted only by a reset endpoint; a payment token should not enter application logs; personal data may require retention and deletion rules. Use allow-listed fields when mapping requests and output only fields the client needs. See [Chapter 131](../09-http-and-application-development/131-authorization.md) for object authorization and [Chapter 143](./143-input-validation.md) for validation boundaries.

```php
<?php

declare(strict_types=1);

final readonly class SecurityContext
{
    public function __construct(
        public int $subjectId,
        public int $tenantId,
        public string $requestId,
    ) {
    }
}

function requireTenant(SecurityContext $context, int $resourceTenantId): void
{
    if ($context->tenantId !== $resourceTenantId) {
        throw new RuntimeException('Resource is unavailable');
    }
}
```

The exception message sent to a client should be a safe problem response; the detailed reason belongs in a protected, redacted log. Enforce the same boundary in HTTP handlers, command-line jobs, queue consumers, exports, and administrative tools.

## Secrets and Dependencies

Treat passwords, signing keys, database credentials, provider tokens, and encryption keys as secrets with a lifecycle: creation, distribution, rotation, use, revocation, and destruction. Keep them out of source control, images, URLs, exception messages, and ordinary logs. A secret manager can reduce exposure, but the application still needs scoped permissions and a rotation strategy.

Give each service account the smallest database and network permissions that support its task. Separate read and write credentials where practical. Pin and audit dependencies, verify webhook signatures, validate provider responses, and bound every network call with a timeout. A dependency outage must produce a defined failure or degraded mode; it must not silently grant access or discard an audit event.

## Logging, Auditing, and Detection

Logs support diagnosis; audit records support accountability. Record a correlation ID, actor or service identity, action, resource class and safe identifier, outcome, and time according to privacy policy. Never log passwords, bearer tokens, session cookies, reset links, full payment data, or unredacted request bodies by default.

Security events include repeated authentication failures, permission changes, token use, exports, support impersonation, secret rotation, and anomalous volume. An audit record should be durable and access-controlled, and its absence should be observable. Do not make a remote logging service a reason to block every request indefinitely; choose a bounded failure policy for each event.

Detection is part of the control. Alert thresholds must account for normal traffic, preserve enough context to investigate, and avoid exposing sensitive content in the alert channel. Practice the response: revoke credentials, contain access, preserve evidence, notify affected parties when required, and restore from a known-good state.

## Failure and Threat Analysis

* **Fail-open authorization:** a missing policy response grants access. Default to denial and surface a retryable error.
* **Confused deputy:** a privileged worker accepts an unscoped user instruction. Carry the subject, tenant, and authorized action in the command.
* **Trust-boundary drift:** a new webhook, export, or queue consumer bypasses the normal middleware. Test every entry point.
* **Secret leakage:** debug context or metrics include credentials. Use structured redaction and secret-scanning checks.
* **Availability attack:** expensive parsing, queries, or provider calls consume all workers. Apply size limits, timeouts, quotas, and backpressure.
* **Stale authorization:** cached roles or memberships outlive a revocation. Version or invalidate policy data.
* **Missing evidence:** a sensitive action succeeds without an audit event. Define atomicity, retries, and reconciliation for the audit path.

## Testing and Assurance

Test security properties at the boundary where they matter. Include anonymous, authenticated, wrong-tenant, expired, disabled, and privileged subjects. Test malformed input, oversized payloads, replay, concurrent transitions, dependency timeouts, and logging redaction. Use integration tests with the real authorization query and database constraints; unit tests alone cannot detect an unscoped route lookup.

Run dependency and secret scanning in CI, review high-risk changes with a threat model, and exercise incident response. Security headers and cookie attributes need browser or protocol tests. The [OWASP ASVS](https://github.com/OWASP/ASVS) provides a versioned set of verification requirements that can become project acceptance criteria.

## Exercises

1. Draw trust boundaries for a PHP API with a browser, queue, database, cache, object store, and payment provider. State one assumption and one control at each boundary.
2. Threat-model an account email-change flow using STRIDE. Assign a test or operational signal to each high-risk threat.
3. Define a security event schema for a tenant data export. Include safe identifiers, outcome, retention, and redaction rules.
4. Review an endpoint that calls an external provider. List its timeout, retry, signature, credential, and fail-open decisions.

## Review Questions

1. What makes a boundary trusted or untrusted in an application?
2. How do confidentiality, integrity, and availability differ?
3. Why is defense in depth stronger when controls are independent?
4. What is a confused deputy in a queue or service-to-service flow?
5. Which data should be excluded from ordinary logs?
6. Why do security controls need monitoring and recovery procedures?

## Summary

A security model names assets, actors, trust boundaries, threats, controls, and evidence. Apply secure defaults and defense in depth across HTTP, PHP, data stores, queues, dependencies, and operations. Authenticate and authorize explicitly, validate and encode at the correct boundary, minimize secret exposure, log safely, detect abuse, and test the properties that protect real data and actions.

## References

- [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [PHP Configuration Runtime Configuration](https://www.php.net/manual/en/configuration.php)
- [NIST SP 800-61 Rev. 3: Incident Response Recommendations](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
