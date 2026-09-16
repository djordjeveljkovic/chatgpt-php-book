# AI Summary — Chapter 145 — XSS

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains stored/reflected/DOM XSS, context-specific encoding, safe DOM APIs, CSP, cookie layers, testing, exercises, and review questions.

## Concepts already explained

Output encoding is sink-specific; HTML, JavaScript, CSS, URL, and rich-HTML contexts need different controls. CSP and `HttpOnly` reduce impact but do not replace correct encoding or authorization.

## Terminology established

XSS, stored XSS, reflected XSS, DOM XSS, output sink, context-specific encoding, CSP, safe DOM API.

## Examples used

PHP `htmlspecialchars()` output and safe DOM/CSP policies.

## Cross-references

- [Chapter 134 — APIs](../../volumes/09-http-and-application-development/134-apis.md)
- [Chapter 146 — CSRF](../../volumes/10-security/146-csrf.md)

## Exact next section

Chapter 146 — CSRF: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated security proofread. XSS guidance links to OWASP, PHP, and MDN.
