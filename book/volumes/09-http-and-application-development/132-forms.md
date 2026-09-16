---
book: The Complete Modern PHP Engineering Book
volume: 9
volume_title: HTTP AND APPLICATION DEVELOPMENT
chapter: 132
title: Forms
slug: forms
status: complete
summary: ../../_ai/chapter-summaries/132-forms-summary.md
---

# Chapter 132 — Forms

## Why This Matters

A form is an untrusted message that asks the application to change state. Browser controls improve usability, but a client can omit fields, alter hidden values, replay a request, or send a different content type. Reliable form handling validates at the server boundary, protects the request against cross-site request forgery, preserves useful errors, and applies the change atomically.

## Mental Model

```text
render form + server token
        ↓
browser submits untrusted fields
        ↓
parse → authenticate → authorize → validate
        ↓
commit domain change → redirect
```

The form representation is not the domain model. Convert validated input into a command or value object before calling business code. Never pass the raw `$_POST` array through the application.

## Validation and Normalization

Validate presence, type, length, range, and relationships between fields. Normalize only where the domain says values are equivalent. Trimming a display name may be reasonable; silently changing an account identifier can hide an error. Keep error messages separate from the values so a failed submission can be rendered without losing the user's safe input.

```php
<?php

declare(strict_types=1);

function validateProfile(array $input): array
{
    $name = trim((string) ($input['name'] ?? ''));
    $email = filter_var($input['email'] ?? null, FILTER_VALIDATE_EMAIL);
    $errors = [];

    if ($name === '' || mb_strlen($name) > 100) {
        $errors['name'] = 'Enter a name no longer than 100 characters.';
    }
    if ($email === false) {
        $errors['email'] = 'Enter a valid email address.';
    }

    return $errors === []
        ? ['name' => $name, 'email' => strtolower((string) $email)]
        : throw new InvalidArgumentException('Invalid profile form');
}
```

Use domain validation after shape validation. A syntactically valid email can still violate a uniqueness constraint or an account policy. Database constraints remain authoritative under concurrent requests.

## CSRF and Method Semantics

A state-changing browser request should carry a CSRF token tied to the user's session or a signed request context. Compare tokens in constant time, expire them according to the session policy, and reject missing or invalid values. `SameSite` cookies reduce some cross-site requests but are not a substitute for server validation when the application supports browser sessions.

Use `POST` or another state-changing method for mutations. A successful mutation should normally return a redirect (Post/Redirect/Get) so refreshing the result page does not resubmit the form. The redirect target must be server-controlled or validated to prevent an open redirect.

## Authorization and Idempotency

CSRF answers “did this request come from an allowed browser context?” Authorization answers “may this actor perform this operation?” Check authorization after authenticating the actor and before changing state; never trust a hidden `user_id` or `role` field.

If a user can double-click or a browser retries, make the command idempotent where the operation permits it. A unique request key or domain constraint can turn a repeated submission into the original result rather than a duplicate row.

## Security and Testing

Escape values for the output context when rendering errors or previous input. HTML escaping is different from SQL parameter binding and URL encoding. Do not echo an uploaded filename, rich text, or error message into HTML without the correct context-specific escaping.

Test missing fields, malformed types, boundary lengths, cross-field rules, invalid CSRF tokens, unauthorized actors, duplicate submissions, and database constraint failures. Test the redirect and that a failed request does not partially commit a change.

## Common Mistakes

- Trusting browser-side `required`, `pattern`, or hidden fields.
- Validating shape but not authorization or domain rules.
- Reusing raw input as HTML without context-specific escaping.
- Mutating state on `GET`.
- Returning a successful form response that refreshes into a duplicate submission.
- Treating a CSRF token as an authorization decision.

## Senior Engineer Thinking

Forms are a boundary adapter. Keep parsing, validation, CSRF, authorization, domain execution, and presentation distinct. The controller should make the order visible and should return a response that gives the user a safe next action after both success and failure.

## Exercises

1. Design a profile form command with field errors and a domain-level uniqueness error.
2. Add a session-backed CSRF token and test missing, wrong, and valid tokens.
3. Convert a mutating `GET` endpoint into Post/Redirect/Get with an idempotency key.

## Review Questions

1. Why is browser validation insufficient?
2. What does CSRF protection establish, and what does authorization establish?
3. Why should a successful form mutation redirect?
4. Which constraints still belong in the database?

## Summary

Treat form fields as untrusted input. Validate and normalize at the boundary, protect browser mutations against CSRF, authorize the actor, apply the command atomically, escape output for its context, and redirect after success. Tests should cover malformed input, security failures, retries, and persistence errors.

## References

- [OWASP: Cross-Site Request Forgery Prevention](https://owasp.org/www-community/attacks/csrf)
- [PHP Manual: `filter_var`](https://www.php.net/manual/en/function.filter-var.php)
- [MDN: HTTP method semantics](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)
