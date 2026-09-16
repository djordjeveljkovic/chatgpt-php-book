# AI Summary — Chapter 259 — Secrets

- Status: complete
- Volume: Volume 17 — PRODUCTION ENGINEERING
- Last updated: 2026-09-16

## Written material

Explains secret inventory, ownership, storage and delivery, narrow secret handles, signing-key rotation, expiry and revocation, logging, PHP worker retention, testing, security, and exposure response. Includes SecretProvider and ApiSigner examples.

## Concepts already explained

Secret lifecycle, blast radius, least-privilege delivery, secret handle, rotation overlap, revocation, exposure response, and redaction surface.

## Terminology established

Secret inventory, active credential, verification overlap, key ID, propagation time, emergency replacement, and protected observation path.

## Examples used

Secret inventory, typed SecretProvider and ApiSigner, key-rotation timeline, DSN/log exposure, worker refresh, and rotation drills.

## Cross-references

Chapters 155, 254, and 258.

## Open threads

Continue with structured production evidence in Chapter 260.

## Exact next section

Chapter 260 — Logging: the Why This Matters section.

## Technical verification notes

PHP 8.5.10 linted the chapter's PHP examples. Local Markdown links resolved and git diff --check passed. No real secret, manager, or rotation integration was used.

## Writing notes

Treats secret protection as control of the full lifecycle and every observation path.
