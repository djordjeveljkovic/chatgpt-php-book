---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 146
title: CSRF
slug: csrf
status: complete
summary: ../../_ai/chapter-summaries/146-csrf-summary.md
---

# Chapter 146 — CSRF

## Why This Matters

Cross-site request forgery abuses ambient browser credentials. A malicious site causes a victim's browser to send a request to an application where the victim is already signed in, and the browser attaches its session cookie automatically. The attack targets state-changing operations.

## Defenses

Use a server-generated unpredictable token tied to the session or request context, validate it on every browser mutation, and compare it safely. `SameSite` cookies, origin checks, and checking `Sec-Fetch-Site` can add defense in depth, but do not replace a deliberate token policy for high-risk actions.

Do not use `GET` for mutations. For APIs using bearer tokens in an explicit header rather than cookies, traditional cookie CSRF is different, but token theft, CORS, and object authorization remain concerns.

## PHP Boundary

```php
<?php

declare(strict_types=1);

function verifyCsrf(string $expected, ?string $provided): void
{
    if ($provided === null || !hash_equals($expected, $provided)) {
        throw new RuntimeException('Invalid request token');
    }
}
```

Generate tokens with `random_bytes()`, keep them out of URLs when possible, rotate them with session policy, and avoid logging them. A token proves possession of the browser's server-issued context; it does not authorize the requested action.

## Testing and Common Mistakes

Test missing, wrong, expired, cross-session, and replayed tokens; state-changing methods; content types; login/logout transitions; and cookie attributes. Common mistakes include disabling CSRF for convenience, validating only JavaScript headers, accepting a token from a cookie without an independent value, and assuming `SameSite` covers every browser and deployment path.

## Senior Engineer Thinking

CSRF is a browser credential-attachment problem. Choose credentials and transport deliberately, require an independent request proof for cookie-authenticated mutations, and layer origin, cookie, and authorization controls.

## Exercises

1. Add a session token to a profile form and test invalid submissions.
2. Compare cookie-authenticated and bearer-header API requests.
3. Define SameSite and origin policies for a cross-site payment flow.

## Review Questions

1. Why does the browser attach a session cookie to a forged request?
2. What does a CSRF token prove and not prove?
3. Why are `GET` mutations dangerous?

## Summary

Protect cookie-authenticated state changes with unpredictable server-validated tokens and safe method semantics. Use SameSite and origin signals as layered defenses, and keep CSRF separate from authentication and authorization.

## References

- [OWASP: Cross-Site Request Forgery Prevention](https://owasp.org/www-community/attacks/csrf)
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PHP Manual: `hash_equals`](https://www.php.net/manual/en/function.hash-equals.php)
