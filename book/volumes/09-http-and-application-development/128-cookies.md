---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 128
title: Cookies
slug: cookies
status: complete
summary: ../../_ai/chapter-summaries/128-cookies-summary.md
---

# Chapter 128 — Cookies

## Why This Matters

A cookie is a small piece of browser-managed state attached to HTTP requests. It can remember a language preference, carry a session identifier, or participate in a login flow. The server does not get to treat it as trusted storage: the browser can omit it, replay it, and sometimes expose it to script or another origin when the cookie policy permits.

Cookies therefore sit at several boundaries at once. They are response headers, request input, browser policy, and often a key to server-side state. A sound design chooses what the browser needs to remember, limits where the value travels, and assumes that a stolen value can be replayed until it expires or is revoked.

## The Request and Response Model

The server asks a browser to store a cookie with `Set-Cookie`:

```http
Set-Cookie: theme=dark; Path=/; Max-Age=2592000; Secure; HttpOnly; SameSite=Lax
```

On a later matching request, the browser sends a `Cookie` request header:

```http
Cookie: theme=dark
```

The browser decides whether a cookie matches by considering its host, path, lifetime, and security attributes. The server should not assume that a cookie will be present, unique, fresh, or syntactically valid. Multiple cookies with the same name can exist under different paths or domains, which makes broad cookie parsing and deletion error-prone.

`Set-Cookie` is not an ordinary comma-separated list header. Emit one header field per cookie. In PHP, `setcookie()` performs the necessary header encoding and should be called before response output:

```php
<?php

declare(strict_types=1);

setcookie('theme', 'dark', [
    'expires' => time() + 30 * 86400,
    'path' => '/',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Lax',
]);
```

The return value tells you whether PHP accepted the header for emission; it does not tell you that the browser stored it. A proxy, browser policy, invalid attribute, or an already-started response can still prevent the expected result.

## Attributes That Limit Exposure

Use the narrowest policy that satisfies the feature:

| Attribute | Effect and design question |
| --- | --- |
| `Secure` | Sends the cookie over HTTPS only. Use it for credentials and any production application. |
| `HttpOnly` | Prevents normal JavaScript access through `document.cookie`; it does not stop an XSS payload from making authenticated requests. |
| `SameSite` | Controls cross-site sending. `Strict` is strongest, `Lax` commonly supports top-level navigation, and `None` requires `Secure` and should be deliberate. |
| `Path` | Limits request paths. It is routing scope, not an authorization boundary. |
| `Domain` | Makes a cookie available to a host and, when specified, its subdomains. Omit it unless subdomain sharing is required. |
| `Max-Age`/`Expires` | Defines persistence. A session cookie still lasts as long as the browser session chooses; it is not a security revocation mechanism. |

For a host-only session cookie, omit `Domain`, use `Path=/`, and use `Secure`, `HttpOnly`, and an appropriate `SameSite` value. A `__Host-` cookie name strengthens this convention: it must be `Secure`, have `Path=/`, and have no `Domain` attribute. A `__Secure-` name requires `Secure` but permits a domain. These prefixes are browser-enforced conventions; they do not replace server-side checks.

Cookies have practical size and count limits that vary by user agent. Store a short opaque identifier rather than a serialized account, permission set, or large JSON document. Never put a password, bearer token with no revocation plan, or sensitive personal data into a client-controlled value merely because it can be encoded or signed.

## Values Are Input

Cookie values are not integrity-protected by default. Base64 is an encoding, not encryption or authentication. If a client needs to carry a value that the server can verify without a lookup, use an authenticated construction with a key kept outside the cookie. A typical format is `payload.signature`, where the signature covers the exact payload and is checked in constant time. Include an expiry and a purpose in the payload so a value for one feature cannot be replayed as another.

For credentials, a random opaque identifier mapped to server-side state is usually easier to revoke and rotate. If a cookie is signed, key rotation and replay remain operational concerns: a valid stolen cookie remains valid until its expiry or the server rejects its key/version.

## A Small Preference Cookie

Preferences are a good cookie use because losing one is harmless. Validate the value against an allow-list and fall back safely:

```php
<?php

declare(strict_types=1);

function readTheme(array $cookies): string
{
    $theme = $cookies['theme'] ?? '';

    return in_array($theme, ['light', 'dark'], true) ? $theme : 'light';
}

function writeTheme(string $theme, bool $https): void
{
    if (!in_array($theme, ['light', 'dark'], true)) {
        throw new InvalidArgumentException('Unknown theme');
    }

    setcookie('theme', $theme, [
        'expires' => time() + 30 * 86400,
        'path' => '/',
        'secure' => $https,
        'httponly' => true,
        'samesite' => 'Lax',
    ]);
}
```

An application behind a TLS-terminating proxy must determine HTTPS from trusted proxy configuration, not from an arbitrary `X-Forwarded-Proto` request header. For a credential cookie, prefer a fixed production configuration that always enables `Secure`; local HTTP development can use a separate environment setting.

To delete a cookie, send an expired cookie with the same name, domain, and path used to create it:

```php
setcookie('theme', '', [
    'expires' => 1,
    'path' => '/',
    'secure' => true,
    'httponly' => true,
    'samesite' => 'Lax',
]);
```

Deleting the browser value does not revoke server-side state. Logout and credential revocation must invalidate the associated session or token as well.

## Failure and Threat Analysis

* **Cookie theft:** malware, an exposed browser profile, a proxy mistake, or an XSS vulnerability can reveal a value. Use HTTPS, `HttpOnly`, short lifetimes for sensitive sessions, rotation, and server-side revocation.
* **Session fixation:** an attacker may cause a victim to use a known session identifier. Generate or rotate the identifier after authentication; see [Chapter 129](129-sessions.md).
* **Cross-site request forgery:** a cookie can be attached automatically to a cross-site request. Use `SameSite` as a browser-layer control and a CSRF token or equivalent request proof for state-changing actions.
* **Subdomain compromise:** a domain cookie can be sent to every eligible subdomain. Keep sensitive services on separate hosts or use host-only/`__Host-` cookies.
* **Header injection:** never concatenate unvalidated request data into a `Set-Cookie` header. Use `setcookie()` and validate names and values.
* **Replay and concurrency:** a valid cookie can be copied and sent twice or from multiple locations. Make sensitive operations idempotent where needed and record security events without logging the raw value.

`HttpOnly` reduces one theft path but does not make the application safe from XSS, CSRF, or a compromised device. Treat it as one layer in a complete design.

## Testing Cookies

Unit-test the value policy without depending on global headers: pass a cookie array into a reader and assert that unknown, malformed, and missing values use the safe default. At the HTTP boundary, use a response abstraction that records headers and assert the complete `Set-Cookie` attributes. Browser tests should verify that JavaScript cannot read `HttpOnly` cookies, HTTPS is required for `Secure`, cross-site navigation has the intended behavior, and logout expires the matching path.

Test proxy and deployment configuration separately. A test that runs only on plain HTTP can accidentally bless a configuration that browsers will reject in production. Never print cookie values in test failure output or application logs.

## Exercises

1. Implement a `CookieJar` boundary that reads an allow-listed theme and emits a host-only preference cookie. Test missing, invalid, and valid values.
2. Find every cookie in a sample application. For each, record its purpose, lifetime, `Secure`, `HttpOnly`, `SameSite`, `Domain`, and `Path` settings, then justify any broad scope.
3. Add a browser test showing that changing a cookie's `Path` prevents deletion when the deletion path differs. Explain why the server must retain cookie metadata.
4. Design a signed, expiring preference value. Specify the key rotation and replay behavior, then decide whether a server-side opaque value would be simpler.

## Review Questions

1. Why is a cookie value request input even when the server originally issued it?
2. What does `HttpOnly` protect, and what does it not protect?
3. Why should a session cookie usually omit `Domain`?
4. Why are `Base64` and URL encoding insufficient for integrity or confidentiality?
5. Why must cookie deletion use the original path and domain?
6. What operational problem remains after signing a cookie?
7. How do `SameSite` and a CSRF token provide different controls?

## Summary

Cookies are browser-managed request state. Scope them narrowly, use `Secure`, `HttpOnly`, and an intentional `SameSite` policy for credentials, validate every value, and prefer short opaque identifiers over application data. A cookie policy must account for theft, fixation, replay, subdomain exposure, deployment proxies, and the difference between deleting browser state and revoking server state.

## References

- [PHP `setcookie()` manual](https://www.php.net/manual/en/function.setcookie.php)
- [PHP session security ini settings](https://www.php.net/manual/en/session.security.ini.php)
- [RFC 6265, HTTP State Management Mechanism](https://www.rfc-editor.org/rfc/rfc6265)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
