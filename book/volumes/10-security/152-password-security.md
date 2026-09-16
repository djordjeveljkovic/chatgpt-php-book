---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 152
title: Password Security
slug: password-security
status: complete
summary: ../../_ai/chapter-summaries/152-password-security-summary.md
---

# Chapter 152 — Password Security

## Why This Matters

A password is a user-chosen secret that the application must verify without being able to recover it. Password security includes hashing, login behavior, reset and recovery, breached-password handling, rate limits, multi-factor options, and the operational controls around every credential flow.

A strong hash cannot repair an insecure reset endpoint, an account-enumeration leak, or a session that remains valid after a password change. Treat the whole credential lifecycle as one security boundary.

## Hash, Do Not Encrypt

Store a password hash produced by a password-specific, adaptive algorithm. Encryption is reversible and requires protecting a decryption key; password hashing is intentionally one-way and can be made expensive for offline guessing. PHP's password API stores the algorithm, cost, and salt in the hash string:

~~~php
<?php

declare(strict_types=1);

function hashPassword(string $password): string
{
    if (strlen($password) < 12) {
        throw new InvalidArgumentException('Password is too short');
    }

    $hash = password_hash($password, PASSWORD_DEFAULT);
    if ($hash === false) {
        throw new RuntimeException('Could not hash password');
    }

    return $hash;
}

function verifyPassword(string $password, string $hash): bool
{
    return password_verify($password, $hash);
}
~~~

Never build a salt scheme by hand, hash with a fast general-purpose digest such as SHA-256 alone, or truncate the password before hashing. PASSWORD_DEFAULT may change as PHP gains a better default; use password_needs_rehash() after a successful verification and replace the stored hash. The password API handles salt generation and constant-time verification for its supported algorithms.

The cost is a capacity decision. A high cost slows attackers but also consumes login workers. Benchmark on production-like hardware, set a maximum login workload, and change parameters gradually. A login endpoint must remain available under attack; rate limits, queues, and account protections complement the hash cost.

## Password Policy and User Experience

Prefer long passwords and passphrases, allow password managers and paste, and reject known-compromised passwords when the product can check them without sending the clear password to a third party. Avoid arbitrary composition rules and silent truncation: they produce predictable substitutions and surprise users. Compare policy requirements with the account's recovery and MFA risk rather than treating a local length check as a proof of safety.

Normalize the account identifier used for lookup according to a documented policy. Do not normalize the password itself. A password containing leading or trailing spaces must be treated as the user entered it. Store no password in logs, analytics, support notes, email, or exception context.

## Login and Enumeration

A failed lookup and a wrong password should produce the same visible response and similar timing. When no account exists, applications can verify against a fixed dummy hash to avoid an obvious timing difference. Rate-limit by account and network signals, but do not let an attacker lock every account by sending failures for other users.

After a successful password verification, rotate the authenticated session identifier. If the account has a stronger factor, complete the factor challenge before granting the full session. Notify users of password changes and provide a revocation path for existing sessions and remember-me tokens.

## Reset and Recovery

A reset link is a temporary bearer credential. Generate it with random_bytes(), store only a hash with an account, purpose, creation time, and expiry, consume it once, and invalidate older tokens after success. The request response should not disclose whether an email address is registered. Rate-limit email delivery and make a reset request safe to repeat without generating an unbounded flood.

Recovery channels are part of the account's security level. Email access, recovery codes, support-assisted recovery, and device approval have different proof strengths. Require recent authentication or step-up verification before changing a password, email, MFA factor, or recovery method. Record security events without storing reset URLs or codes.

## MFA and Breach Response

MFA reduces the risk of a stolen password, but enrollment and recovery can become bypasses. Protect factor enrollment with recent authentication, confirm a new factor, store recovery codes securely, and revoke or replace them after use. Prefer phishing-resistant authenticators where the threat model supports them.

If a credential database or provider is compromised, disable or reset affected credentials, invalidate sessions and long-lived tokens as appropriate, investigate access, communicate with users, and preserve evidence. Do not quietly lower hash cost or keep using a known-exposed password set.

## Failure and Threat Analysis

* **Offline cracking:** a leaked database enables guessing. Use an adaptive password hash, unique salts from the API, and breach response.
* **Credential stuffing:** passwords reused elsewhere succeed. Add MFA, rate limits, breached-password checks, and suspicious-login detection.
* **Enumeration:** different login or reset messages reveal accounts. Use an equivalent public response.
* **Reset theft:** links leak through logs or referrers. Use HTTPS, short expiry, one-time use, and safe landing pages.
* **Session persistence:** old sessions remain valid after a password change. Track and revoke sessions.
* **Recovery bypass:** weak support or email recovery defeats MFA. Apply a documented proof and audit policy.

## Testing Password Flows

Test hashing, verification, rehashing, long passwords, wrong passwords, disabled accounts, generic errors, rate-limit boundaries, and session rotation. Use a fake clock and repository in unit tests; do not assert a fixed hash string. Integration tests should prove reset tokens are single-use and hashed at rest, expired tokens fail, reset requests do not enumerate accounts, and password changes revoke the selected credentials.

## Exercises

1. Design a password table and reset-token table, including hash fields, expiry, revocation, and audit data.
2. Benchmark password verification at several costs and define an operational ceiling for login capacity.
3. Threat-model a support-assisted password reset and list the evidence required before completion.
4. Add a regression test proving that a password change invalidates an old session and remember-me token.

## Review Questions

1. Why should passwords be hashed rather than encrypted?
2. What does password_needs_rehash() enable?
3. Why should password input not be normalized like an email identifier?
4. How can reset requests leak account existence?
5. Why are factor enrollment and recovery part of password security?
6. Which credentials should be invalidated after a password change?

## Summary

Password security covers the full credential lifecycle. Use PHP's password API, keep password input intact, rate-limit and make failures resistant to enumeration, hash one-time reset tokens, rotate sessions, protect MFA recovery, and plan for breached credentials. Test the flow across hashing, reset, recovery, revocation, and operational capacity.

## References

- [PHP password_hash()](https://www.php.net/manual/en/function.password-hash.php)
- [PHP password_verify()](https://www.php.net/manual/en/function.password-verify.php)
- [PHP password_needs_rehash()](https://www.php.net/manual/en/function.password-needs-rehash.php)
- [PHP random_bytes()](https://www.php.net/manual/en/function.random-bytes.php)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

