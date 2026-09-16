---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 153
title: Session Security
slug: session-security
status: complete
summary: ../../_ai/chapter-summaries/153-session-security-summary.md
---

# Chapter 153 — Session Security

## Why This Matters

A browser session turns a successful authentication into a credential presented on later requests. Whoever controls the session identifier can often act as the user until it expires or is revoked. Session security therefore covers cookie transport, fixation, theft, cross-site requests, storage, rotation, expiry, logout, and behavior when the session store is unavailable.

The session ID is an opaque handle. It should not contain authorization claims that the application trusts without checking current server-side state.

## Cookie Protections

Use a cookie configured for the deployment's transport and browser threat model. HTTPS-only transport requires Secure. HttpOnly prevents ordinary JavaScript from reading the cookie, which limits some theft paths but does not prevent an attacker from making requests through an XSS payload. SameSite reduces cross-site sending and supports CSRF defense, but it is not a replacement for an explicit CSRF policy for state-changing requests.

~~~php
<?php

declare(strict_types=1);

function sendSessionCookie(string $sessionId): void
{
    setcookie('app_session', $sessionId, [
        'expires' => 0,
        'path' => '/',
        'secure' => true,
        'httponly' => true,
        'samesite' => 'Lax',
    ]);
}
~~~

Choose Domain narrowly, preferably omitting it so the cookie is host-only. Use a __Host- prefix when the deployment can meet its rules: Secure, Path /, and no Domain attribute. A cross-site embedding or federated login may require a different SameSite policy; document the flow and test it in real browsers.

## Fixation, Rotation, and Privilege Changes

Session fixation happens when an attacker causes a victim to use an identifier known to the attacker and the application later grants that identifier authenticated privileges. Start a new session or rotate the identifier after login, MFA completion, password reset, and privilege elevation. Preserve only the minimal intended state.

~~~php
<?php

function establishAuthenticatedSession(int $userId): void
{
    if (!session_regenerate_id(true)) {
        throw new RuntimeException('Could not rotate session');
    }

    $_SESSION = [
        'user_id' => $userId,
        'authenticated_at' => time(),
        'auth_level' => 'password',
    ];
}
~~~

Rotation must be coordinated with concurrent requests. A request already in flight may use the old ID; mark it revoked or use a server-side session record with a rotation version when the threat model requires strict invalidation. Do not copy attacker-controlled session values into the new session.

## Server-Side State and Expiry

Store only an opaque ID in the browser and keep authentication state server-side. Session records should include a user, creation time, last activity, expiry, authentication level, device or risk metadata according to privacy needs, and a revocation marker. Use idle and absolute timeouts based on the sensitivity of the action. A session that is inactive for an hour may still be too powerful after a month.

A session store is a security dependency. If Redis or the database is unavailable, a protected endpoint should fail closed or provide a clear reauthentication/retry response. Do not treat a missing record as a new anonymous session and accidentally continue an administrative flow. Protect session data from cross-tenant key collisions and apply access control to the store.

## Logout and Revocation

Logout should revoke the server-side record and expire the browser cookie. Clearing a cookie alone does not invalidate a stolen copy. Provide account-wide revocation after password changes, suspicious activity, or an administrator action. Remember-me tokens are a separate credential and need separate revocation.

A distributed application may receive a request on a different node immediately after logout. Use a shared session store or a revocation/version mechanism with a defined consistency window. A short cache of an authorization decision can outlive a revocation; include versioning or invalidate it on sensitive changes.

## CSRF, XSS, and Transport

Cookie authentication is automatically attached by the browser, which creates cross-site request forgery risk. Use SameSite as one layer, a CSRF token bound to the session for state-changing browser requests, and origin or referer checks where appropriate. Do not exempt an endpoint merely because it returns JSON. APIs using an Authorization header have a different browser attachment model, but still require authorization and replay controls.

HttpOnly cannot stop XSS from performing actions as the user. Output-encode in the correct context, enforce a content security policy where appropriate, avoid dangerous HTML sinks, and keep session cookies out of page scripts. TLS must cover the complete route, including redirects and internal proxy hops where credentials could otherwise be exposed.

## Session Lifecycle and Observability

Set a session ID using PHP's secure session facilities or a cryptographically secure random source; never derive it from a user ID, timestamp, or predictable counter. Do not put session IDs in URLs, logs, analytics, error messages, or referrer-visible pages. Redact cookie headers in request logging.

Measure active sessions, rotation failures, invalid-session attempts, revocations, and store latency. Alert on unusual geographic or device changes only with privacy-aware policy and a response path. Security telemetry must not become another credential store.

## Failure and Threat Analysis

* **Fixation:** the identifier survives authentication. Rotate it at every privilege boundary.
* **Theft:** a token leaks through transport, logs, XSS, or a device. Use TLS, HttpOnly, redaction, short lifetimes, and revocation.
* **CSRF:** a browser attaches a cookie to an unwanted request. Use SameSite and a server-checked CSRF token.
* **Store outage:** the application cannot determine identity. Fail closed for protected actions.
* **Logout gap:** only the browser cookie is cleared. Revoke server-side state and related tokens.
* **Subdomain exposure:** a broad Domain cookie is sent to a less-trusted host. Prefer host-only or __Host- cookies.
* **Concurrent rotation:** old requests race with a new session. Define a rotation and revocation policy.

## Testing Session Security

Use browser or protocol tests for Secure, HttpOnly, SameSite, path, and host-only behavior. Test session ID rotation at login, MFA, password changes, and role elevation. Verify old IDs cannot authorize after revocation, logout invalidates a copied ID, expiry is enforced, and a session-store failure cannot grant access. Add CSRF tests for every browser state change and XSS-context tests for user-controlled output.

## Exercises

1. Define a session record and cookie policy for a banking-style application. Explain idle, absolute, and step-up expiry.
2. Write a test that captures an anonymous session ID, logs in, and proves the authenticated ID is different and the old ID is unusable.
3. Model logout across two application nodes with a shared store and with a revocation version. State the consistency trade-off.
4. Review every place your application logs request headers and identify session identifiers that must be redacted.

## Review Questions

1. Why does HttpOnly reduce risk without preventing XSS actions?
2. What is session fixation, and when should an ID rotate?
3. Why does clearing a cookie not revoke a stolen session?
4. What should a protected endpoint do when the session store is unavailable?
5. How do SameSite and CSRF tokens complement each other?
6. Why is a broad cookie Domain a security concern?

## Summary

A session identifier is a credential. Keep it opaque and server-side, use Secure, HttpOnly, and an appropriate SameSite policy, rotate it at authentication and privilege changes, enforce idle and absolute expiry, revoke server-side state on logout, and fail closed when the store is unavailable. Pair cookie controls with CSRF protection, output encoding, TLS, redacted logs, and tests that exercise real browser behavior.

## References

- [PHP Sessions](https://www.php.net/manual/en/book.session.php)
- [PHP session_regenerate_id()](https://www.php.net/manual/en/function.session-regenerate-id.php)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [MDN Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie)

