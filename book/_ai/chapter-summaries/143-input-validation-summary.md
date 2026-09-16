# AI Summary — Chapter 143 — Input Validation

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains semantic parsing, shape/type/range validation, normalization, output-context separation, parser limits, trusted commands, testing, exercises, and review questions.

## Concepts already explained

Validation establishes acceptable input; it does not replace output encoding, authorization, SQL binding, or database constraints. Reject malformed values, normalize deliberately, and bound parser resources.

## Terminology established

Semantic parser, normalization, trusted command, parser limit, output context, unknown-field policy.

## Examples used

A strict positive-ID parser and boundary policies for JSON, dates, money, and identifiers.

## Cross-references

- [Chapter 142 — Security Model](../../volumes/10-security/142-security-model.md)
- [Chapter 144 — SQL Injection](../../volumes/10-security/144-sql-injection.md)

## Exact next section

Chapter 144 — SQL Injection: the Why This Matters section.

## Technical verification notes

PHP examples and local links are covered by the consolidated security proofread. Validation guidance links to OWASP and the PHP Manual.
