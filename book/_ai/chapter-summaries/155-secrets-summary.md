# AI Summary — Chapter 155 — Secrets

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains secret inventory and ownership, storage and delivery, least privilege, short-lived credentials, rotation overlap, redaction, incident response, testing, exercises, and review questions.

## Concepts already explained

Secrets need a lifecycle of creation, scoped access, use, rotation, revocation, and recovery. Environment variables are delivery mechanisms rather than vaults; logging, backups, images, and process tooling are exposure paths.

## Terminology established

Secret inventory, secret manager, least privilege, short-lived credential, rotation overlap, revocation, redaction.

## Examples used

PHP required-secret loading, an authenticated HTTP header, and a two-key signing rotation reader.

## Cross-references

- [Chapter 142 — Security Model](../../volumes/10-security/142-security-model.md)
- [Chapter 153 — Session Security](../../volumes/10-security/153-session-security.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 156 — Dependency Security: the Why This Matters section.

## Technical verification notes

PHP examples and local links were linted in the consolidated security proofread. Secret guidance links to OWASP, PHP, and NIST documentation.
