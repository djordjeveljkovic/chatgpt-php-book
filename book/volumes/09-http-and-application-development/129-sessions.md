---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 129
title: Sessions
slug: sessions
status: complete
summary: ../../_ai/chapter-summaries/129-sessions-summary.md
---

# Chapter 129 — Sessions

## Why This Matters

HTTP does not remember the previous request. A session adds continuity by putting a random session identifier in a cookie and keeping mutable state on the server. This lets an application remember a signed-in user, a shopping cart, or a one-time flash message without placing all of that state in the browser.

A session is not an authentication system by itself. It is a state-management mechanism whose identifier becomes a bearer credential when it identifies a logged-in account. The design must therefore cover identifier quality, fixation, rotation, expiration, storage failure, concurrent requests, and logout.

## The Two Parts of a PHP Session

PHP's session extension separates the identifier from the data. `session_start()` reads the session identifier from a configured cookie, loads data through a session save handler, and populates `$_SESSION`. On shutdown, PHP writes changed session data back through the handler.

The default file handler is convenient for one host, but a load-balanced deployment needs shared storage or a deliberate routing strategy. Redis, a database, or another handler can provide shared state, though each introduces availability, serialization, eviction, and operational concerns. The session identifier should remain opaque; the browser should never be able to select an account by putting an account ID into it.

Configure security settings before starting the session:

```php
<?php

declare(strict_types=1);

function startSession(bool $https): void
{
    ini_set('session.use_only_cookies', '1');
    ini_set('session.use_strict_mode', '1');
    ini_set('session.use_trans_sid', '0');

    session_set_cookie_params([
        'lifetime' => 0,
        'path' => '/',
        'secure' => $https,
        'httponly' => true,
        'samesite' => 'Lax',
    ]);

    if (session_status() !== PHP_SESSION_ACTIVE && !session_start()) {
        throw new RuntimeException('Could not start session');
    }
}
```

`session.use_strict_mode` asks PHP to reject an uninitialized identifier rather than accepting it and creating state under an attacker-supplied value. It is a defense in depth measure; the application still needs to rotate the identifier after a privilege change. Set the cookie name and other options centrally rather than allowing individual controllers to create conflicting session policies.

## Lifecycle and Rotation

A typical lifecycle is:

1. Create an anonymous session only when the application needs one.
2. Store minimal state, such as a cart identifier or a CSRF token.
3. After successful authentication, call `session_regenerate_id(true)` and associate the new session with the user.
4. On logout or account compromise, remove sensitive state, destroy the server record, and expire the browser cookie.
5. Expire idle or absolute sessions according to the risk of the application.

Regeneration changes the identifier; it does not automatically make every concurrent request safe. A request that started with the old identifier may still be writing when authentication rotates it. Applications with high-value sessions should define how concurrent requests are handled, use a session store with safe writes, and avoid storing large mutable objects in a session.

```php
<?php

function signInSession(int $userId): void
{
    if (session_status() !== PHP_SESSION_ACTIVE) {
        throw new LogicException('Session must be active');
    }

    session_regenerate_id(true);
    $_SESSION = [
        'user_id' => $userId,
        'authenticated_at' => time(),
    ];
}

function signOutSession(): void
{
    $_SESSION = [];

    if (ini_get('session.use_cookies')) {
        $params = session_get_cookie_params();
        setcookie(session_name(), '', [
            'expires' => 1,
            'path' => $params['path'],
            'domain' => $params['domain'] ?: '',
            'secure' => (bool) $params['secure'],
            'httponly' => (bool) $params['httponly'],
            'samesite' => $params['samesite'] ?: 'Lax',
        ]);
    }

    session_destroy();
}
```

The application may keep a small revocation or session-version record for forced logout across devices. Destroying one session record does not revoke another device's session unless the account-level policy says so.

## Session Fixation and Hijacking

Fixation happens when an attacker establishes or predicts an identifier and persuades a victim to authenticate with it. Rotate on login, privilege elevation, and other identity changes. Hijacking happens when an attacker obtains a valid identifier, for example through malware, an exposed log, an insecure transport, or an XSS vulnerability. Use HTTPS, `Secure` and `HttpOnly` cookies, strict cookie scope, short lifetimes where appropriate, and anomaly detection. Do not log raw session IDs, URLs containing them, or complete cookie headers.

Do not rely on an IP address or user-agent string as a session secret. Mobile networks, proxies, and browser updates change these values; an attacker can often imitate them. They can be signals for risk analysis, but a mismatch should be handled as a policy decision rather than as cryptographic proof.

## Concurrency and Storage

PHP's file session handler commonly locks a session while a request has it open. This prevents two requests from blindly overwriting each other's changes, but it also means a slow request can block parallel AJAX requests from the same browser. Read-only endpoints can release the lock after reading:

```php
<?php

session_start();
$userId = $_SESSION['user_id'] ?? null;
session_write_close();

// Perform slow work without holding the session lock.
```

After `session_write_close()`, writes to `$_SESSION` are no longer persisted for that request. Do not use this pattern when later code must update session state. For a shared store, understand whether its locking is per key, how locks expire, what happens when a worker dies, and whether serialization is compatible across PHP versions and deployments.

Session storage is a dependency. If Redis is unavailable, deciding between denying access, showing an anonymous response, or using a bounded fallback is a product and security decision. Failing open for an authenticated action can become an authorization vulnerability. Instrument session-start failures, lock wait time, record size, expiry, and store latency without recording identifiers.

## Flash Data and Minimal State

Sessions are useful for one-request messages, but keep the protocol explicit:

```php
<?php

function flash(string $message): void
{
    $_SESSION['_flash'] = $message;
}

function consumeFlash(): ?string
{
    $message = $_SESSION['_flash'] ?? null;
    unset($_SESSION['_flash']);

    return is_string($message) ? $message : null;
}
```

Treat all session data as potentially stale after a deploy, logout, account change, or key rotation. Store IDs and small values, then load authoritative records from the database. A session should not become a second database with an unbounded cart, permissions snapshot, or object graph.

## Expiration and Recovery

Use both idle and absolute limits for sensitive sessions. Enforce expiration on the server using timestamps stored in the session or session record, not only by the browser cookie lifetime. Clock differences and a failed cleanup job should not turn an expired browser value into authorization.

When a session store loses data, users may be signed out. That is usually safer than guessing an identity from stale client state. Build a clear retry or reauthentication path, and distinguish a temporary dependency failure from an invalid session in logs and metrics. A session store outage can become a denial of service even when the application code is correct.

## Testing Sessions

Test the session boundary with a fake handler or isolated store. Verify that an unknown identifier is rejected in strict mode, login changes the identifier, logout clears data and expires the cookie, expired sessions cannot authorize a request, and malformed values result in an anonymous state. Add a concurrency test that sends two requests for one session and observes the intended locking or merge behavior.

Integration tests should exercise the real reverse proxy and shared session store configuration. Test a store timeout, eviction, serialization mismatch, and deployment with two application workers. Security tests should assert that logs and error pages do not contain session identifiers.

## Exercises

1. Implement a session-backed flash-message flow and test that a message is consumed once even when the next request fails.
2. Measure the latency of five parallel requests sharing one PHP session. Explain the effect of the session lock and redesign the endpoint if it holds the lock while doing I/O.
3. Add login and logout tests that capture the `Set-Cookie` headers and prove that the identifier changes after login and is expired after logout.
4. Design a deployment using two PHP-FPM hosts. Compare shared Redis sessions with sticky routing, including failure and data-loss behavior.

## Review Questions

1. Why is a session identifier a credential after login?
2. What does strict session mode defend against?
3. Why must a session be regenerated after authentication?
4. Why can a session lock create application latency?
5. What state belongs in a session, and what belongs in the database?
6. Why are IP and user-agent checks weak substitutes for a session secret?
7. What should an application do when the session store is unavailable?

## Summary

A PHP session is server-side state addressed by an opaque cookie value. Configure secure cookies and strict mode before `session_start()`, rotate the identifier after authentication, destroy state on logout, limit lifetime and size, and understand the locking and failure behavior of the chosen store. Sessions provide continuity; they do not remove the need for authentication, authorization, CSRF protection, or operational monitoring.

## References

- [PHP Sessions manual](https://www.php.net/manual/en/book.session.php)
- [PHP session security ini settings](https://www.php.net/manual/en/session.security.ini.php)
- [PHP `session_regenerate_id()` manual](https://www.php.net/manual/en/function.session-regenerate-id.php)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
