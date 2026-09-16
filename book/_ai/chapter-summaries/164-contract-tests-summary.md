# AI Summary — Chapter 164 — Contract Tests

- Status: complete
- Volume: Volume 11 — TESTING
- Last updated: 2026-09-16

## Written material

Contract tests make compatibility between an independent consumer and provider executable. The chapter covers focused HTTP contracts, provider states, error and authorization behavior, event and webhook contracts, compatibility classification, tooling, CI, and failure diagnosis.

## Concepts already explained

Consumer-driven contract, provider verification, provider state, schema versus behavior, compatibility window, event contract, and contract ownership.

## Examples used

A focused JSON order contract and a PHP assertion for its stable response fields.

## Cross-references

- [Chapter 162 — API Tests](../../volumes/11-testing/162-api-tests.md)
- [Chapter 136 — Webhooks](../../volumes/09-http-and-application-development/136-webhooks.md)
- [Chapter 140 — API Versioning](../../volumes/09-http-and-application-development/140-api-versioning.md)

## Open threads

Continue with Chapter 165 on test doubles and collaborator boundaries.

## Exact next section

Chapter 165 — Test Doubles: the Why This Matters section.

## Technical verification notes

PHP fences and local links were checked after writing; live Pact/provider verification was not run.
