# AI Summary — Chapter 98 — PSR-4

- Status: complete
- Volume: Volume 7 — COMPOSER AND THE PHP ECOSYSTEM
- Last updated: 2026-09-15

## Written material

Substantial chapter explains the PSR-4 name-to-path contract, Composer's namespace-prefix mappings and package-root-relative paths, exact case and underscore rules, multiple base directories, normal lookup misses, generated-loader rebuild boundaries, deployment/testing practice, security boundaries, exercises, and review questions.

## Concepts already explained

- PSR-4 replaces a namespace prefix with a base directory; namespace suffix components become directories and the final class name becomes a `.php` file.
- Composer's `autoload.psr-4` paths are relative to the root of the package that declares them. Non-empty Composer keys end in a namespace separator; JSON escapes backslashes, while an empty prefix is a Composer fallback extension.
- Underscores are ordinary characters. Namespace/file casing must follow the PSR-4 contract, and case-insensitive development environments can hide failures.
- Composer supports one or more base directories for a prefix. Duplicate class ownership and empty-prefix fallbacks need deliberate boundaries.
- Composer's loader tries the longest matching non-empty prefix first, checks that prefix's base directories in order, then shorter prefixes, and finally empty-prefix fallback directories. This order is Composer-specific rather than a PSR-4 requirement.
- A lookup miss is normal and should not throw; errors inside a found source file are a separate failure.
- Adding a class beneath an existing default PSR-4 mapping does not require regeneration. Adding/changing a mapping does; optimized/authoritative modes need their own build-time tests.
- Autoloading executes source code but does not authorize dynamically selected classes.

## Terminology established

PSR-4, fully qualified class name, namespace prefix, base directory, package root, namespace suffix, lookup miss, fallback prefix, case-sensitive path contract, generated autoloader.

## Examples used

- `Acme\\Billing\\` mapped to `src/`, with `Invoice` and nested `Report\\CsvReport` examples.
- Root-project layout and Composer JSON mapping with escaping.
- Mapping-resolution table, multi-directory configuration, Composer bootstrap and CLI smoke-test commands.
- Verification checklist for default, optimized, installed-package, and case-sensitive environments.

## Cross-references

- [Chapter 92 — Composer](../../volumes/07-composer-and-the-php-ecosystem/092-composer.md)
- [Chapter 94 — `composer.json`](../../volumes/07-composer-and-the-php-ecosystem/094-composer-json.md)
- [Chapter 95 — `composer.lock`](../../volumes/07-composer-and-the-php-ecosystem/095-composer-lock.md)
- [Chapter 97 — Autoloading](../../volumes/07-composer-and-the-php-ecosystem/097-autoloading.md)
- [PHP-FIG PSR-4 specification](https://www.php-fig.org/psr/psr-4/)
- [Composer autoload documentation](https://getcomposer.org/doc/04-schema.md#autoload)
- [Composer `ClassLoader` source](https://github.com/composer/composer/blob/main/src/Composer/Autoload/ClassLoader.php)

## Open threads

No open chapter work remains.

## Exact next section

Chapter 102 — Formatting: the Why This Matters section.

## Technical verification notes

Normative naming, prefix, path-case, underscore, and autoloader error requirements were checked against the PHP-FIG PSR-4 specification. Composer prefix syntax, package-root path rules, multiple base directories, empty-prefix extension, generated mapping location, lookup order, and add-class/regenerate behavior were checked against current official Composer schema, basic usage, optimization docs, and `ClassLoader` source. Composer 2.10.3 passed `composer validate --strict` on the complete minimal root manifest; PHP 8.5.10 linted both example classes and the Composer inline smoke test loaded the mapped class. A new class under an existing prefix loaded without regeneration, and a wrong-case request missed in a separate fresh process. All local links resolved and `git diff --check` passed. Independent proofreading clarified the empty-prefix exception and distinguished Composer's prefix lookup order from the PSR-4 standard.

## Writing notes

Chapter 97 explains SPL registration, Composer's general loading strategies, and autoloader optimization. Chapter 94 covers manifest schema and JSON field syntax. Chapter 98 owns the detailed PSR-4 mapping contract; keep dependency resolution, PHP-FIG process, and broader PSR catalog topics in Chapters 93, 99, and 100.
