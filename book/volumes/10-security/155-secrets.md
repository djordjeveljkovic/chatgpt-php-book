---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 155
title: Secrets
slug: secrets
status: complete
summary: ../../_ai/chapter-summaries/155-secrets-summary.md
---

# Chapter 155 — Secrets

A secret is data whose disclosure gives an attacker a capability: a password, API token, signing key, encryption key, database credential, or recovery code. Secrets are different from ordinary configuration because they require controlled access, limited exposure, rotation, revocation, and evidence of use.

Keeping a value out of Git is necessary, but it is not the complete policy. A secret can leak through a container image, shell history, process arguments, crash dump, debug page, trace, log, backup, dependency, or a developer's local machine.

## Why this matters

A leaked database password can bypass the application’s validation and authorization. A leaked signing key can make forged sessions or webhooks appear legitimate. A leaked provider token may incur charges or grant access to another system. The impact depends on what the credential can do, so scope and expiry are part of secret security.

Start with an inventory. For every secret, record its owner, purpose, permitted audience, storage location, access policy, rotation method, expiry, revocation path, and recovery procedure. A secret without an owner or revocation path is an incident waiting for a trigger.

## Secret versus configuration

A port, feature flag, or public service URL is configuration. A database password, private key, session-signing key, and OAuth client secret are secrets. The distinction is about the consequence of disclosure, not whether a value appears in a file named `.env`.

Do not put real secrets in source control, example fixtures, Docker layers, URLs, ordinary logs, or exception messages:

```php
// Vulnerable: the value can appear in source, history, images, and code review.
$client = new ApiClient('sk_live_real_value');

// Also risky: query strings commonly reach proxy, browser, and analytics logs.
$url = 'https://api.example.test/export?token=' . $token;
```

Use a secret provider or an injected runtime value, and send credentials in the protocol's intended header or body:

```php
<?php

declare(strict_types=1);

function requiredSecret(string $name): string
{
    $value = getenv($name);
    if ($value === false || $value === '') {
        throw new RuntimeException("Required secret is unavailable: {$name}");
    }

    return $value;
}

$apiToken = requiredSecret('BILLING_API_TOKEN');
$context = stream_context_create([
    'http' => [
        'method' => 'POST',
        'header' => "Authorization: Bearer {$apiToken}\r\nContent-Type: application/json\r\n",
        'content' => json_encode(['event' => 'invoice.created'], JSON_THROW_ON_ERROR),
        'timeout' => 5,
    ],
]);
```

The example demonstrates failing closed when a required value is missing. In a real client, prefer a maintained HTTP library, avoid placing tokens in exception text, and verify that the request library does not log headers by default. The secret name may appear in an error; the secret value must not.

Environment variables are a delivery mechanism, not a vault. They can be exposed by process inspection, debugging, crash reporting, child processes, or deployment tooling. On hosts where a secret manager is available, fetch the value at startup or through a short-lived access flow, restrict the process identity, and decide what happens when the manager is unavailable. Do not print the environment while diagnosing a deployment.

## Access and least privilege

Give each application, worker, migration job, and CI task a distinct identity. A read-only reporting worker should not use the production write credential. A deployment job should not receive every runtime token. Scope credentials by database, resource, operation, tenant, and network where the provider supports it.

Keep the secret manager policy separate from the application authorization policy. The application may decide which user can request a report; the runtime identity decides whether the process can read the provider credential. Both decisions must hold.

Use short-lived credentials and workload identity where possible. If a long-lived secret is unavoidable, reduce its scope and set an owner, expiration review, and emergency revocation test. Do not confuse encryption at rest with access control: a secret is still exposed to every process that can decrypt it.

## Rotation without an outage

Rotation is a protocol between issuers, consumers, and operators. Before changing a key, know which versions are active, how old tokens are validated, and how to revoke the old value. For signing keys, publish or retain the old verification key for a bounded overlap while issuing with the new key. For API credentials, create the replacement, deploy consumers that can use it, test, revoke the old one, and verify that no traffic still depends on it.

A two-key reader can support a controlled transition:

```php
<?php

declare(strict_types=1);

function activeSigningKeys(): array
{
    $keys = [];
    foreach (['APP_SIGNING_KEY_CURRENT', 'APP_SIGNING_KEY_PREVIOUS'] as $name) {
        $value = getenv($name);
        if (is_string($value) && $value !== '') {
            $keys[] = $value;
        }
    }

    if ($keys === []) {
        throw new RuntimeException('No signing key is configured');
    }

    return $keys;
}
```

A verifier should identify the key version in an authenticated envelope or try only a bounded, configured set. Remove the previous key after the maximum token lifetime and clock-skew window. Rotation without revocation is only replacement; it does not invalidate a stolen credential immediately.

## Logging, redaction, and failure

Never log passwords, bearer tokens, private keys, session cookies, reset links, or full authorization headers. Redaction must happen before serialization because a structured logger may copy the value into multiple fields. Define a safe representation such as a key identifier, provider name, or last four characters only when that still cannot be used as a credential.

Avoid putting secrets in exception messages and metrics labels. Scrub command output, HTTP traces, queue payloads, and support exports. Test redaction with realistic nested arrays and objects, including failure paths. A log sink that is unavailable should follow a bounded policy; do not silently drop the audit evidence of a key change.

If a secret is exposed, preserve evidence without copying the value into more systems. Revoke or rotate it, identify the time window and permissions, inspect use logs, remove it from accessible artifacts where feasible, and record the incident and follow-up controls. Git history rewriting does not revoke a credential.

## Testing and operations

Test that production startup fails when a required secret is absent, that unauthorized identities cannot read it, and that rotation works before and after the overlap window. Scan commits, artifacts, images, logs, crash reports, and configuration snapshots with tools suited to the environment. Treat scanner findings as a trigger for verification and rotation, not as proof that a value is safe.

Use secret-manager access logs, unusual token use alerts, expiry dashboards, and periodic access reviews. Backups and replicas are part of the secret's lifecycle. Decide whether encrypted backups retain a key after account deletion and how the key itself is recovered without creating a new uncontrolled copy.

## Exercises

1. Inventory the secrets in a PHP web process, queue worker, migration job, and CI pipeline. Assign a separate identity and rotation owner to each.
2. Design a signing-key rotation with a seven-day token lifetime and five minutes of clock skew. State when the previous key can be revoked.
3. Write a logger test proving that nested authorization headers and reset links are redacted on both success and exception paths.

## Review questions

- Why is an environment variable not a complete secret-management solution?
- How do scope, expiry, and revocation reduce the impact of a leak?
- What must happen before revoking the old credential during rotation?
- Which values should never appear in ordinary logs or metric labels?
- Why does deleting a secret from Git history not remediate a leak?

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [PHP manual: getenv](https://www.php.net/manual/en/function.getenv.php)
- [NIST SP 800-57 Part 1 Rev. 5: Key Management](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-4/sp800-63b.html)
