# AI Summary — Chapter 100 — PSR Standards

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

Catalogs PHP-FIG recommendations by domain and practical use, snapshots current accepted/draft/deprecated/abandoned statuses, describes the evolving Coding Style PER, and gives package-boundary adoption, testing, maintenance, and operational guidance. Includes a logger/clock injection example, exercises, and review questions.

## Concepts already explained

- A PSR is a voluntary provider-consumer contract, not a PHP runtime feature, implementation, or operational guarantee.
- Accepted standards in the catalog cover coding conventions (PSR-1/12), autoloading (PSR-4), logging (PSR-3), caching (PSR-6/16), containers (PSR-11), HTTP messages/links/handlers/factories/clients (PSR-7/13/15/17/18), events (PSR-14), and clocks (PSR-20).
- PSR-6 exposes hit state; PSR-16's simple `get()` cannot distinguish a cached `null` from a miss.
- PSR-11 standardizes `get()`/`has()`, not autowiring or application DI structure. PSR-18 treats valid HTTP 4xx/5xx as responses. PSR-20 returns wall-clock `DateTimeImmutable`, not monotonic durations.
- Snapshot from the official index on 2026-09-15: PSR-5/19/21/22 Draft; PSR-0/2 Deprecated; PSR-8/9/10 Abandoned. PER Coding Style 3.1 is evolving and says it extends/replaces PSR-12 while requiring PSR-1.
- Adoption requires reading normative prose, checking package/runtime compatibility, testing provider and consumer behavior, and retaining explicit application security/reliability policy.

## Terminology established

Provider-consumer contract, accepted PSR, draft, deprecated recommendation, abandoned proposal, evolving PER, public package boundary, conformance test, wall clock.

## Examples used

- Domain/status catalog of all currently accepted PSRs and selected current proposal statuses.
- `InvoiceAudit` consuming `Psr\\Log\\LoggerInterface` and `Psr\\Clock\\ClockInterface` with an injectable timestamp.
- Boundary-focused contract-test examples for logging, cache misses, HTTP client errors, clocks, and server middleware.

## Cross-references

- [Chapter 98 — PSR-4](../../volumes/07-composer-and-the-php-ecosystem/098-psr-4.md)
- [Chapter 99 — PHP-FIG](../../volumes/07-composer-and-the-php-ecosystem/099-php-fig.md)
- [Chapter 101 — Static Analysis](../../volumes/07-composer-and-the-php-ecosystem/101-static-analysis.md)
- [Chapter 102 — Formatting](../../volumes/07-composer-and-the-php-ecosystem/102-formatting.md)
- [PHP-FIG PSR status index](https://www.php-fig.org/psr/)
- [PHP-FIG PER index](https://www.php-fig.org/per/)

## Open threads

No chapter-specific open threads. Recheck the status snapshot if editing after 2026-09-15.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

PSR and PER status labels were checked on 2026-09-15 against the PHP-FIG primary [PSR index](https://www.php-fig.org/psr/) and [PER index](https://www.php-fig.org/per/). Scope and key behavior summaries were checked against the corresponding PHP-FIG specifications, including PSR-1, 3, 6, 7, 11–18, 20, PSR-12, and PER Coding Style 3.1. Current proposal URLs were followed from the official index to confirm their destinations. PHP 8.5.10 linted the example code; all relative Markdown links in the chapter and summary resolve, and `git diff --check` passes. Independent proofreading confirmed the status catalog and specification summaries with no necessary corrections. The PHP example is illustrative; its use of standard interfaces remains an application integration boundary rather than a self-contained runtime test.

## Writing notes

Chapter 99 covers PHP-FIG governance and lifecycle mechanics. Chapter 98 owns PSR-4 path rules. Keep static-analysis tooling in Chapter 101 and formatter use/configuration in Chapter 102.
