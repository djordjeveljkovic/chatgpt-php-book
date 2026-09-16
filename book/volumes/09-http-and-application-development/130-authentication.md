---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 130
title: Authentication
slug: authentication
status: complete
summary: ../../_ai/chapter-summaries/130-authentication-summary.md
---

# Chapter 130 — Authentication

## Why This Matters

Authentication answers, “Who is making this request?” Authorization answers, “May that identity perform this action?” Mixing the two leads to code that identifies a user but fails to protect a resource, or grants access based on an unverified client claim.

Authentication is a protocol, not a login form. It covers how an account is enrolled, how a credential is presented, how failures are handled, how an authenticated browser is represented, and how access is revoked. The strongest implementation chooses a credential appropriate to the threat model and keeps the proof separate from application data.

## Passwords Are Verifiers, Not Secrets to Decrypt

The server should store a password hash produced by a password-hashing algorithm, never the password or a reversible encryption of it. PHP's `password_hash()` chooses a supported password algorithm and parameters; `password_verify()` checks a submitted password against the stored hash without the application handling a salt separately.

```php
<?php

declare(strict_types=1);

function createPasswordHash(string $password): string
{
    if (strlen($password) < 12) {
        throw new InvalidArgumentException('Use a longer password');
    }

    $hash = password_hash($password, PASSWORD_DEFAULT);
    if ($hash === false) {
        throw new RuntimeException('Password hashing failed');
    }

    return $hash;
}

function verifyPassword(string $password, string $storedHash): bool
{
    return password_verify($password, $storedHash);
}
```

The hash string contains the algorithm and cost metadata, so a future change can be detected with `password_needs_rehash()` after a successful login:

```php
if (password_verify($password, $account->passwordHash)) {
    if (password_needs_rehash($account->passwordHash, PASSWORD_DEFAULT)) {
        $account->replacePasswordHash(createPasswordHash($password));
    }
    // Rotate the authenticated session here.
}
```

Choose password length and breached-password policy with the product's risk and recovery flow. A local minimum does not make a password safe if the same password was exposed elsewhere. Do not silently truncate passwords, impose arbitrary composition rules that encourage predictable patterns, or send passwords to logs, analytics, email, or support tickets.

## A Login Boundary

The login service should accept a credential, fetch an account by a normalized login identifier, verify the password, and return an authenticated identity. A failed lookup and a wrong password should have the same externally visible result. Otherwise an attacker can enumerate registered accounts. The code must also enforce disabled, unverified, locked, and password-reset states according to the account policy.

```php
<?php

final class LoginService
{
    public function __construct(
        private AccountRepository $accounts,
        private SessionAuthenticator $sessions,
        private LoginRateLimiter $rateLimiter,
    ) {
    }

    public function login(string $email, string $password, string $bucket): void
    {
        $normalizedEmail = mb_strtolower(trim($email), 'UTF-8');
        $this->rateLimiter->assertAllowed($bucket);
        $account = $this->accounts->findByEmail($normalizedEmail);

        $valid = $account !== null
            && $account->isEnabled()
            && password_verify($password, $account->passwordHash());

        if (!$valid) {
            $this->rateLimiter->recordFailure($bucket);
            throw new InvalidCredentials();
        }

        $this->rateLimiter->recordSuccess($bucket);
        $this->sessions->authenticate($account->id());
    }
}
```

In a real implementation, use a dummy hash verification when no account exists if timing differences matter for the deployment. Rate limiting should combine signals such as account and network while avoiding a policy that lets an attacker lock every account by submitting failures. Return a generic response and send detailed reasons to protected security logs.

## Sessions, Tokens, and Remember Me

For a browser application, a server-side session with a secure cookie is often the simplest authenticated representation. Rotate it at login and privilege elevation; see [Chapter 129](129-sessions.md). For an API, a bearer token may be appropriate, but any party holding it can present it. TLS protects it in transit; it does not make an unexpired token revocable or safe to place in a browser-accessible location.

“Remember me” needs its own design. Use a random, high-entropy selector/token pair, store only a hash of the token server-side, associate it with an account and device record, expire it, and rotate it after use. If a rotated token is presented again, revoke the token family and require a full login: this can detect theft. Do not store a password-equivalent long-lived token in a database in plaintext.

```php
<?php

function issueRememberToken(int $accountId): string
{
    $selector = bin2hex(random_bytes(9));
    $validator = bin2hex(random_bytes(32));
    $validatorHash = hash('sha256', $validator);

    // Persist selector, validatorHash, accountId, expiry, and token-family ID.
    saveRememberToken($selector, $validatorHash, $accountId, time() + 30 * 86400);

    return $selector . ':' . $validator;
}
```

The cookie carrying this value must use the same transport and browser protections as a session credential. Revoke all remember-me records on password reset or suspected compromise, and do not expose a token in a URL where it can enter history, referrer headers, or analytics.

## Stronger Factors and Delegated Identity

Passwords are vulnerable to reuse, phishing, and database compromise. Multi-factor authentication adds a separate proof, such as a time-based one-time password, a hardware security key, or a platform authenticator. Recovery codes are credentials too: hash them where practical, show them once, and revoke or replace them after use.

WebAuthn/passkeys bind authentication to a registered public key and origin. OAuth 2.0 is an authorization framework; OpenID Connect adds an identity layer. When using an external identity provider, validate the issuer, audience/client, signature, nonce/state, redirect URI, token expiry, and account-linking policy. Do not treat an email address in an arbitrary access token as proof of identity. Follow the provider's current documentation and use a maintained library for protocol details.

## Account Recovery and Verification

Password reset links are temporary bearer credentials. Generate them with `random_bytes()`, store only a hash with an account and expiry, use them once, and invalidate older tokens after a successful reset. Send the link over HTTPS and avoid placing secrets in referrer-prone pages. The response should not reveal whether an email address belongs to an account.

Email verification, changing an email address, adding a factor, and disabling a factor are authentication events. Require recent authentication or step-up verification for high-impact changes. Record event type, account, timestamp, outcome, and safe contextual data; protect logs from becoming a source of credentials or personal-data leakage.

## Failure and Threat Analysis

* **Credential stuffing:** rate-limit and monitor failures, support password managers, and check passwords against breach data where policy permits.
* **Enumeration:** make registration, login, and reset responses equivalent for existing and nonexistent accounts.
* **Session fixation:** rotate the session after login and role changes.
* **Phishing and token theft:** use MFA or passkeys, short-lived access, secure cookies, and revocation paths.
* **Reset abuse:** hash reset tokens, expire and consume them once, and notify the account through a trusted channel.
* **Account linking:** require proof of control of both identities before merging external and local accounts.
* **Availability attacks:** avoid an unlimited password-cost increase, unbounded reset emails, or a rate limiter that attackers can use to lock all accounts.

Authentication failure must fail closed for access decisions. A dependency failure, such as an unavailable identity provider or session store, should produce a clear retry or reauthentication path; it should not turn an unknown identity into an authenticated one.

## Testing Authentication

Unit-test password verification, rehashing, normalization, disabled accounts, generic failures, expiry, and single-use reset tokens. Use a fake clock and deterministic repositories in unit tests; never assert a password hash's exact string because salts and cost parameters can change.

Integration tests should prove that a successful login rotates the session, an old session cannot authorize the account, logout revokes access, and a remember-me token rotates or is rejected on replay. Test rate-limit boundaries and reset-email idempotency. Browser or protocol tests should exercise the real redirect and callback validation for an external provider, including an invalid state, issuer, audience, nonce, and expired token.

## Exercises

1. Build a login service around a repository and session gateway. Add tests for unknown account, wrong password, disabled account, success, and password rehash.
2. Design a remember-me token table. Include token-family rotation, expiry, password-reset revocation, and replay detection.
3. Threat-model password reset and email-change flows. List every bearer value, where it can leak, and how it is invalidated.
4. Compare passwords plus TOTP with passkeys for a consumer application. Include recovery, support, account sharing, and phishing considerations.

## Review Questions

1. Why should passwords be hashed instead of encrypted?
2. What does `password_needs_rehash()` enable?
3. Why should a missing account and wrong password have the same response?
4. Why is a bearer token difficult to revoke safely?
5. What must an OIDC callback validate before linking an account?
6. Why should reset tokens be stored as hashes?
7. Which events deserve step-up authentication?

## Summary

Authentication establishes identity through a credential and a lifecycle. Hash passwords with PHP's password API, make failures resistant to enumeration and abuse, rotate browser sessions after login, design remember-me and reset tokens as revocable one-time credentials, and validate every assertion from an external identity provider. MFA and passkeys reduce password risk, but recovery and account linking remain part of the authentication protocol.

## References

- [PHP `password_hash()` manual](https://www.php.net/manual/en/function.password-hash.php)
- [PHP `password_verify()` manual](https://www.php.net/manual/en/function.password-verify.php)
- [PHP `random_bytes()` manual](https://www.php.net/manual/en/function.random-bytes.php)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
