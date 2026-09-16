---
book: The Complete Modern PHP Engineering Book
volume: 21
volume_title: REFERENCE
chapter: 307
title: Security Checklist
slug: security-checklist
status: complete
summary: ../../_ai/chapter-summaries/307-security-checklist-summary.md
---

# Chapter 307 — Security Checklist

## Why This Matters

A security checklist is useful only when each control has an owner and evidence. Review the whole trust boundary, not just the controller: FPM, CLI, workers, exports, search, webhooks, migrations, repair tools, dependencies, configuration, logs, and recovery all process authority or sensitive data.

## Identity and Authorization

- [ ] authentication identifies the principal and handles expiry, replay, and session security;
- [ ] authorization checks the requested action against the actual resource;
- [ ] tenant scope is derived from trusted context and enforced in queries and writes;
- [ ] list, search, export, cache, job, webhook, CLI, and repair paths have negative tests;
- [ ] policy changes while work is queued have an explicit rule;
- [ ] failures do not disclose another tenant or sensitive resource.

Hidden buttons, client-provided tenant IDs, opaque object IDs, and a successful login are not authorization.

## Inputs and Sinks

- [ ] input is parsed and bounded before use;
- [ ] validation is separated from output encoding and access control;
- [ ] SQL, HTML, shell, path, template, and serialization sinks use the right defense;
- [ ] uploads have size, type, storage, filename, scanning, and retrieval controls;
- [ ] callback URLs validate scheme, destination, redirects, private ranges, DNS behavior, timeout, and response size;
- [ ] parsers and decompression have resource limits.

Signed or authenticated input may still be unsafe to render, store, fetch, or process without policy checks.

## Secrets and Dependencies

- [ ] secrets are absent from source, artifacts, logs, URLs, and exception payloads;
- [ ] secret access is least-privileged, rotated, auditable, and tested;
- [ ] Composer direct/transitive changes, plugins, scripts, advisories, lock file, artifact provenance, and PHP compatibility are reviewed;
- [ ] production and CI extension/runtime differences are known;
- [ ] vulnerable dependency rollback does not silently restore exposure.

## Queues, Webhooks, and External Effects

- [ ] message identity, schema compatibility, authentication, authorization, and replay handling are explicit;
- [ ] duplicate delivery and unknown completion are safe;
- [ ] acknowledgement follows durable completion;
- [ ] webhook signatures, timestamp/replay windows, tenant scope, and idempotency are verified;
- [ ] outbound providers have least-privileged credentials, deadlines, quotas, and reconciliation;
- [ ] callback and export status cannot be used to infer another tenant’s data.

## Logging, Detection, and Abuse

- [ ] structured logs redact tokens, cookies, credentials, personal data, and uploaded content;
- [ ] correlation IDs permit investigation without over-collecting data;
- [ ] retention, access, deletion, and audit policies are defined;
- [ ] alerts cover repeated authorization failures, cross-tenant test failures, abnormal exports, SSRF attempts, dependency drift, and resource exhaustion;
- [ ] rate limits and quotas are scoped by principal, tenant, resource, and operation;
- [ ] controls fail safely when the limiter, audit store, or policy service is unavailable.

## Evidence and Ownership

For every check, record the control, threat, evidence, owner, expiry/review date, and residual risk. Evidence may be a code path, permission matrix, integration test, dependency report, configuration assertion, audit sample, alert exercise, or recovery drill. A scan or green test is a signal, not a complete security claim.

## Release and Recovery

Before release, review changed trust boundaries, schema/message compatibility, secret rotation, migration access, logs, flags, and rollback exposure. During an incident, contain access and data exposure, preserve evidence, revoke or rotate credentials when justified, and verify that recovery does not reintroduce the vulnerability. Rollback can be unsafe if it restores a known security flaw or incompatible data.

## Common Mistakes

- checking only authentication;
- trusting object or tenant IDs;
- omitting jobs, exports, repair tools, and caches;
- logging full requests;
- validating a URL only before redirects;
- treating signatures, scans, or coverage as universal proof;
- ignoring replay and unknown completion;
- rolling back a fix without exposure analysis;
- leaving emergency credentials or debug flags active.

## Exercises

1. Build a permission matrix for a tenant-scoped search and export feature.
2. Review a callback URL flow for redirects, DNS, private ranges, and egress.
3. Trace a secret through configuration, logs, queue messages, and provider calls.
4. Design duplicate/replay tests for a signed webhook.
5. Write a release and recovery record for an authorization fix.

## Review Questions

- Why is authentication insufficient?
- Where must tenant scope travel?
- What makes a signed payload unsafe despite valid cryptography?
- Which evidence supports an authorization claim?
- When can rollback worsen a security incident?
- How should a checklist represent residual risk?

## Summary

Security readiness is evidence across identity, authorization, tenant isolation, inputs, sinks, files, outbound requests, secrets, dependencies, queues, webhooks, logs, abuse controls, release, and recovery. Assign owners, test negative paths, preserve privacy during investigation, and record residual risk.

## Chapter 308 Handoff

Security controls must survive deployment and operation. Chapter 308 closes the book with a production checklist covering artifacts, configuration, health, traffic, workers, capacity, backups, rollback, incidents, and retirement.

## References

- [Chapter 142 — Security Model](../10-security/142-security-model.md)
- [Chapter 157 — Supply-Chain Security](../10-security/157-supply-chain-security.md)
- [Chapter 258 — Configuration](../17-production-engineering/258-configuration.md)
- [Chapter 293 — Security Review](../20-senior-engineering/293-security-review.md)
- [Chapter 308 — Production Checklist](308-production-checklist.md)
