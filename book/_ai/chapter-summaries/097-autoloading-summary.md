# AI Summary — Chapter 97 — Autoloading

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

Explains PHP's class-like autoload mechanism, explicit includes, SPL registration and ordering, Composer's generated bootstrap and autoload strategies, loader failure boundaries, performance/memory trade-offs, security, testing, exercises, and review questions.

## Concepts already explained

- PHP autoloads undefined classes, interfaces, traits, and enums through registered callbacks; functions and constants require explicit or eager file inclusion.
- `spl_autoload_register()` maintains an ordered queue; `prepend` changes order, and an exception interrupts later callbacks.
- Composer builds `vendor/autoload.php` from metadata; the application includes it at bootstrap, then PHP invokes its loader on demand.
- Composer supports PSR-4, PSR-0, classmap, and eager `files` inclusion. Detailed manifest syntax and PSR-4 rules are deferred to Chapters 94 and 98.
- Optimized class maps can reduce lookup work; authoritative maps disable PSR fallback for misses; APCu caches lookup results and consumes APCu memory.

## Terminology established

Autoload callback, autoload queue, class-like symbol, explicit include, generated autoloader, PSR-4, PSR-0, class map, eager file inclusion, optimized class map, authoritative class map, APCu lookup cache.

## Examples used

- Two-file custom SPL autoloader and runtime checks for successful/missing class lookup and function behavior.
- Composer bootstrap using `vendor/autoload.php`.
- Strategy comparison table and development/production optimization trade-offs.

## Cross-references

- [Chapter 92 — Composer](../../volumes/07-composer-and-the-php-ecosystem/092-composer.md)
- [Chapter 94 — `composer.json`](../../volumes/07-composer-and-the-php-ecosystem/094-composer-json.md)
- [Chapter 98 — PSR-4](../../volumes/07-composer-and-the-php-ecosystem/098-psr-4.md)
- [Chapter 59 — OPcache](../../volumes/04-php-under-the-hood/059-opcache.md)

## Open threads

No open chapter work remains.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Verified behavior and strategy descriptions against the official [PHP autoloading manual](https://www.php.net/manual/en/language.oop5.autoload.php), [`spl_autoload_register()` manual](https://www.php.net/manual/en/function.spl-autoload-register.php), [Composer autoload documentation](https://getcomposer.org/doc/01-basic-usage.md#autoloading), [Composer autoloader optimization guide](https://getcomposer.org/doc/articles/autoloader-optimization.md), and [Composer schema](https://getcomposer.org/doc/04-schema.md#autoload). PHP 8.5.10 linted and ran the two-file SPL example; a temporary Composer project passed `dump-autoload` and exercised PSR-4 class loading plus eager `files` inclusion. Independent proofreading found no necessary changes. All 12 local links resolve and `git diff --check` passes.

## Writing notes

Chapter 94 covers `composer.json` field syntax; Chapter 98 owns detailed PSR-4 mapping rules. Keep those details out of this chapter.
