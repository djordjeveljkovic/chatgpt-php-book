---
book: The Complete Modern PHP Engineering Book
volume: 17
volume_title: PRODUCTION ENGINEERING
chapter: 259
title: Secrets
slug: secrets
status: complete
summary: ../../_ai/chapter-summaries/259-secrets-summary.md
---

# Chapter 259 — Secrets

## Why This Matters

A secret is a value whose disclosure or unauthorized use creates harm: a database password, signing key, API token, private certificate, recovery code, or encryption key. Keeping a secret out of Git is necessary but not sufficient. It can still leak through images, environment dumps, command lines, logs, traces, crash reports, support bundles, backups, or overly broad runtime access.

Design the full secret lifecycle: create, store, deliver, use, rotate, revoke, audit, and destroy. The application should know as little as possible and retain a secret for as short a time as the operation permits.

## Inventory and Ownership

For each secret record:

* purpose and owning team;
* system and identity it authenticates;
* allowed readers and environment;
* creation, expiry, rotation, and revocation process;
* blast radius if exposed;
* where it may appear and where it must not appear;
* recovery and emergency replacement procedure.

An inventory exposes forgotten credentials and shared accounts. Prefer separate identities per service and environment. A production database password should not be shared with a developer laptop or a test job.

## Storage and Delivery

Use a secret-management facility or protected file mechanism appropriate to the environment. The delivery path must authenticate the workload and authorize only the needed secret. Mounting every secret into every container is not least privilege.

Environment variables are convenient, but they can appear in process diagnostics, crash dumps, child-process environments, or support tooling. Mounted files can have permission and rotation semantics. A secret manager API adds a network dependency and startup failure mode. Choose deliberately and protect the observation paths.

Never commit secrets, put them in a container image layer, pass them as command-line arguments, or use them as a build artifact. A placeholder configuration file is fine; a real credential is not.

## Secret Handles

Keep secret material at the narrowest boundary. Business code often needs a signed result or authenticated client, not the raw private key:

~~~php
<?php

declare(strict_types=1);

interface SecretProvider
{
    public function read(string $name): string;
}

final readonly class ApiSigner
{
    public function __construct(private string $key)
    {
        if ($key === '') {
            throw new InvalidArgumentException('Empty signing key');
        }
    }

    public function sign(string $payload): string
    {
        return hash_hmac('sha256', $payload, $this->key);
    }
}

function makeSigner(SecretProvider $secrets): ApiSigner
{
    return new ApiSigner($secrets->read('billing-signing-key'));
}
~~~

The provider adapter controls access and error handling. Avoid exposing the key through a general configuration array or a debug representation. In a higher-assurance design, signing occurs inside a key-management service and the application receives only a signature.

## Rotation

Rotation changes a credential while old and new participants may overlap:

```text
publish new key → verifiers accept old and new → migrate signers → revoke old
```

For verification, keep the previous public key or secret version long enough to validate in-flight messages and tokens. For database credentials, create a new identity, deploy consumers, verify usage, then revoke the old identity. Do not rotate only one side of a shared connection without a compatibility plan.

A key identifier makes rotation observable. Include version or key ID in signed envelopes where the protocol permits it, but do not expose private material. Test rollback: a deployment may need the previous verification key even after a new signer is active.

## Revocation and Exposure

Expiry limits exposure; revocation stops a credential before expiry. Define how quickly each system observes revocation. A long-lived token cached in a worker or provider can remain effective after a control-plane change.

If a secret may have leaked, stop ordinary rotation advice and use an incident procedure: revoke, replace, search protected logs and artifacts, identify use, and assess data access. Do not delete evidence before preserving it under the incident policy.

## Logging and Error Handling

Never log raw secret values. Redaction must cover headers, query parameters, bodies, exception context, environment dumps, and serialized objects. Avoid partial secrets that make guessing easier or still violate policy.

An error should say that a credential is unavailable or invalid, not print the value or a complete connection string. Be careful with DSNs: usernames, hosts, and query parameters can also be sensitive.

## PHP Runtime and Workers

PHP variables containing secret material occupy process memory until released and may be copied by application code. Long-running workers can retain them longer than a request. Load at the narrowest point, avoid global caches, overwrite or release references where practical, and recycle workers after rotation when required by the runtime policy.

Environment and process inspection are operating-system boundaries. Restrict diagnostic access and do not include `phpinfo()` or full environment output in public endpoints. See [Chapter 254 — Linux for PHP Engineers](./254-linux-for-php-engineers.md).

## Testing

Test missing secret, denied access, invalid version, rotation overlap, revocation, provider outage, startup failure, worker refresh, and redaction. Use fake values in test fixtures and verify that logs, traces, exceptions, images, and artifacts do not contain them.

Run a controlled rotation drill and measure propagation time, active old-key usage, failed requests, rollback behavior, and revocation completion. Do not print real secrets to prove a test can retrieve them.

## Security

Use least privilege, separate environments, short-lived credentials where practical, strong audit trails, encrypted transport and storage, and protected backup access. A secret manager reduces exposure but becomes part of the availability and authorization model. Cache only when the freshness and revocation contract permits it.

## Common Mistakes

* Treating “not in Git” as the complete secret policy.
* Baking credentials into image layers or build arguments.
* Passing secrets in command-line arguments.
* Giving every service access to every environment's secrets.
* Rotating a signer without verification overlap.
* Forgetting secrets retained in long-running PHP workers.
* Logging a DSN, authorization header, or exception context.
* Revoking a leaked key without preserving incident evidence.

## Senior Engineer Thinking

Ask who can read, use, rotate, revoke, and audit a secret, how long it remains effective, and what evidence is exposed at every boundary. A secret is not secure because one file is chmodded; its entire lifecycle and observation surface must be controlled.

## Exercises

1. Build a secret inventory for database, provider, signing, and encryption credentials.
2. Design a zero-downtime signing-key rotation with verification overlap and rollback.
3. Search a synthetic application image, logs, traces, and crash report for accidental secret exposure.
4. Define the incident steps after a production API token is suspected to be leaked.

## Review Questions

* Which parts of a secret's lifecycle need an owner?
* Why can environment variables and command lines leak credentials?
* What overlap is needed during rotation?
* How can long-running PHP workers retain a revoked secret?
* What is the difference between expiry and revocation?
* Why must incident evidence be preserved after exposure?

## Summary

Secrets require lifecycle control, not merely repository exclusion. Inventory ownership and blast radius, deliver only necessary values to authenticated workloads, keep material behind narrow boundaries, rotate with overlap and rollback, revoke quickly when exposed, redact every observation path, refresh long-running workers, and test retrieval, denial, rotation, revocation, and incident procedures.

## References

- [OWASP: Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [NIST SP 800-57: Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Chapter 155 — Secrets](../10-security/155-secrets.md)
- [Chapter 254 — Linux for PHP Engineers](./254-linux-for-php-engineers.md)
