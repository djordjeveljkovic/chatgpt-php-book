# AI Summary — Chapter 156 — Dependency Security

- Status: complete
- Volume: Volume 10 — SECURITY
- Last updated: 2026-09-16

## Written material

Explains Composer dependency graphs and lockfiles, audit and validation, update review, Composer plugins, advisory response, runtime boundaries, testing, exercises, and review questions.

## Concepts already explained

A dependency runs with application privileges. `composer.lock` improves repeatability but is not proof of safety; audit findings require reachability and impact analysis. Composer plugins execute during installation and need explicit policy.

## Terminology established

Direct dependency, transitive dependency, lockfile, advisory, reachability, Composer plugin, compensating control, production artifact.

## Examples used

Composer validation/audit commands, locked production installation, and a named `allow-plugins` policy.

## Cross-references

- [Chapter 103 — Automated Refactoring](../../volumes/07-composer-and-the-php-ecosystem/103-automated-refactoring.md)
- [Chapter 157 — Supply Chain Security](../../volumes/10-security/157-supply-chain-security.md)

## Open threads

No chapter-specific open threads.

## Exact next section

Chapter 157 — Supply Chain Security: the Why This Matters section.

## Technical verification notes

JSON syntax and local links were checked in the consolidated security proofread. Dependency guidance links to Composer and OWASP documentation.
