---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 97
title: Autoloading
slug: autoloading
status: complete
summary: ../../_ai/chapter-summaries/097-autoloading-summary.md
---

# Chapter 97 — Autoloading

## Why This Matters

Object-oriented PHP code is commonly split across files. Without autoloading, every entry point must include each class file in the right order, including parent classes, interfaces, and collaborators. That works for a tiny script, but the list becomes brittle as a project grows and as dependencies introduce their own classes.

Autoloading lets PHP ask registered callbacks to load a class-like symbol when code first needs it. Composer builds a loader from package metadata and the project’s own configuration. Understanding the boundary between PHP’s mechanism and Composer’s generated mapping helps diagnose missing classes, avoid eager-loading everything, and choose production optimizations without making runtime-generated classes disappear.

## Mental Model

Autoloading connects a symbol request to a file-loading strategy:

```text
code refers to an undefined class-like name
                    ↓
PHP asks registered autoload callbacks in order
                    ↓
one callback finds and includes a matching file
                    ↓
the file declares the requested symbol
                    ↓
PHP continues with the original operation
```

The autoloader does not download a dependency, instantiate an object, or initialize an application service. It gets an opportunity to define a missing class-like symbol. If none of the callbacks defines it, the original class use fails. The PHP Manual describes `spl_autoload_register()` as giving PHP a last chance to load a missing class-like name before an error.

## What PHP Can Autoload

PHP autoloading applies to class-like names: classes, interfaces, traits, and enumerations. It does not automatically load a function or a global constant when code refers to one. A function file must be included explicitly or loaded as part of Composer’s `files` mechanism. That mechanism is eager: its configured files are included when the project includes `vendor/autoload.php`, not on demand when a function is called.

This distinction affects library design. A class can usually be loaded only when it is first used. A file of helper functions has to be loaded before those functions can be called. Keep eager bootstrap files small and predictable; code placed in them runs as part of application startup.

## Explicit Includes and Autoloaders

An explicit `require` names a file and loads it at the point the statement executes. This is clear and suitable for a small script or a deliberate bootstrap where the dependency order is fixed. `require_once` also asks PHP to avoid including the same resolved file more than once. With explicit includes, the caller owns the file path and order.

An autoloader defers that choice until PHP needs a class-like name. The caller names the symbol; the callback applies a lookup policy. This keeps entry points small and works well for larger projects and third-party packages. It also means a name can fail at the point of use because its file was missing, its mapping was wrong, or the file did not declare the expected symbol.

These approaches can coexist. A project may explicitly include a small procedural bootstrap, use a Composer autoloader for classes, and eagerly include a function file. Autoloading is not a reason to hide important startup side effects in files that are difficult to locate.

## Registering a PHP Autoloader

Modern PHP uses `spl_autoload_register()` to add one or more callbacks to the autoload queue. PHP calls the callbacks in queue order when it needs a class-like symbol that is not already defined. The `prepend` parameter can put a callback at the start of the queue; otherwise it is appended. If a callback cannot handle a name, it should return without defining it so that later callbacks can try. An exception thrown by a callback interrupts the queue, so a lookup miss should not be represented by throwing an exception.

The older global `__autoload()` function allowed only one autoloader. It was deprecated in PHP 7.2 and removed in PHP 8.0. New code should register callbacks with SPL instead.

Here is a small example with two files:

```text
autoload-demo.php
src/Report.php
```

`src/Report.php` declares the class when the callback includes it:

```php
<?php

declare(strict_types=1);

namespace App;

final class Report
{
    public function title(): string
    {
        return 'Monthly totals';
    }
}
```

The entry point registers a loader and exercises both a successful and unsuccessful lookup:

```php
<?php

declare(strict_types=1);

use App\Report;

$attempts = [];

spl_autoload_register(
    static function (string $class) use (&$attempts): void {
        $attempts[] = $class;

        if ($class !== Report::class) {
            return;
        }

        require __DIR__ . '/src/Report.php';
    },
);

$report = new Report();

if ($report->title() !== 'Monthly totals') {
    throw new RuntimeException('Unexpected report title.');
}

if (class_exists('App\\MissingType')) {
    throw new RuntimeException('The missing type should not exist.');
}

if (class_exists(Report::class, autoload: false) !== true) {
    throw new RuntimeException('The report class should now be defined.');
}

if (function_exists('App\\formatReport')) {
    throw new RuntimeException('No helper function was loaded.');
}

if ($attempts !== [Report::class, 'App\\MissingType']) {
    throw new RuntimeException('Unexpected autoload attempts.');
}

echo "Autoload checks passed.\n";
```

The callback deliberately handles one class name instead of turning arbitrary input directly into a filesystem path. A general-purpose class-to-path conversion needs a clear namespace or class allow-list and a stable case-sensitive path convention. This tiny example is for understanding the queue; Composer implements the general mapping strategies described below.

`class_exists()` normally permits autoloading; passing `autoload: false` checks only what PHP has already defined. This is useful when tests need to distinguish “defined now” from “could be loaded.” The example also demonstrates that `function_exists()` does not invoke class autoloaders.

## Composer’s Generated Loader

Composer reads autoload metadata from the root project and its dependencies during installation or `composer dump-autoload`. It generates `vendor/autoload.php` and supporting files. The application includes the generated entry point, usually once during bootstrap:

```php
<?php

declare(strict_types=1);

require dirname(__DIR__) . '/vendor/autoload.php';

$report = new App\Report();
```

Including `vendor/autoload.php` registers Composer’s loader with PHP. PHP invokes that loader later when a class-like name is missing. The generated files are build output, not a service contacted for each class lookup. If the project’s autoload configuration changes, regenerate the files with `composer dump-autoload`; field syntax and practical manifest use are covered in [Chapter 94](094-composer-json.md). The end-to-end Composer build/runtime boundary is introduced in [Chapter 92](092-composer.md).

Composer returns its loader instance when `vendor/autoload.php` is required, so a test runner or specialized bootstrap can retain it and add mappings dynamically. Such runtime additions should be deliberate: their behavior can differ from the generated rules used in production.

## Composer’s Loading Strategies

Composer supports several strategies. The project chooses and configures them through `composer.json`; consult [Chapter 94](094-composer-json.md) for the field syntax.

| Strategy | How it finds code | Appropriate use and cost |
| --- | --- | --- |
| PSR-4 | Maps a namespace prefix to one or more base directories, then maps the remaining class name to a relative path. | The common modern choice for organized namespaced code. Detailed mapping rules belong to [Chapter 98](098-psr-4.md). |
| PSR-0 | Uses an older class-name and namespace-to-path convention. | Mainly for legacy packages that have not migrated; new code should generally prefer PSR-4. |
| Class map | Maps fully qualified class names directly to file paths. | Useful for legacy layouts, classes that do not follow one directory convention, and optimized generated maps. The map costs memory proportional to the number of indexed classes. |
| `files` | Includes listed files when Composer’s autoloader is included. | Useful for functions or constants that PHP cannot autoload. It is eager and can add bootstrap work or side effects. |

The strategies are not interchangeable. Class maps answer a known class-name lookup directly. Namespace-based rules keep ordinary source trees easy to edit, because a new class can be found without regenerating the metadata in development. `files` is for code that must be loaded by inclusion rather than by a class-like name.

## Registration Order and Failure Boundaries

Multiple autoloaders can be useful when a project combines Composer with a specialized loader. PHP tries them in order, stopping once the requested symbol becomes defined. A prepended callback runs earlier and can therefore take precedence over a later callback. Avoid two loaders that both claim the same classes; the result then depends on registration order and which file is loaded first.

A loader should make a narrow attempt and return if the requested name is outside its responsibility. Throwing an exception from an autoloader prevents later loaders from running, as the Manual warns. If all callbacks return without defining the requested symbol, the original `new`, `extends`, interface use, or other class-like reference fails with the relevant PHP error. A missing class is not converted into a successful empty value.

The file can fail independently of lookup: it may be missing, contain a parse error, throw while being included, or declare a different class name. These are different failures and should retain enough context in logs or tests to identify the mapping, file, and symbol involved. Do not use `eval()` to turn a dynamic class name into code.

Autoloading also has a supply-chain boundary. A dependency’s PHP files execute with the application process’s privileges once loaded. An eager Composer `files` entry executes during bootstrap even if the corresponding helper is never called. Review dependency and generated-file changes as executable code, keep startup files small, and do not construct class names or file paths from untrusted request input without a strict allow-list.

## Performance and Memory

Autoloading trades an explicit file list for lookup work on first use. A class that PHP has already loaded does not need another disk lookup through the autoload queue. The first lookup may consult several callbacks or namespace prefixes, perform filesystem checks, and include a PHP file. The actual cost depends on the loader, filesystem and OPcache behavior, number of mappings, and the number of unresolved lookups. Count misses and measure representative requests before adding complexity.

Composer’s optimized autoloader builds a class map from PSR-0 and PSR-4 rules. Known classes can then resolve through that map without the same filesystem search, at the cost of generating and loading a map that contains many class names and paths. The map must be generated from the exact code tree being deployed. Run `composer dump-autoload --optimize` as part of the release build rather than copying generated output from another revision.

An authoritative class map treats a class absent from the map as unavailable and skips the PSR fallback search. This is fast for a fixed release, but it can break code that generates classes at runtime or adds files after generation. Test such frameworks and code generators before enabling authoritative mode.

Composer can also use APCu to cache autoload lookup results, including misses. This requires the APCu extension and uses APCu memory; it does not generate a class map on its own. Composer documents authoritative maps and APCu as alternative Level 2 strategies. Choose based on measured lookup behavior and deployment requirements. OPcache can reduce PHP file compilation work, but it does not change which class names your autoloader can resolve; see [Chapter 59](../04-php-under-the-hood/059-opcache.md) for OPcache.

## Testing Autoload Behavior

Autoload behavior should be tested as part of the project bootstrap, not only by checking that JSON parses. Useful checks include:

- load a representative application class from a clean installation and verify its behavior;
- confirm a known missing class is not accidentally resolved by a broad fallback;
- verify that helper functions are available only after the intended explicit or Composer `files` include;
- run the same bootstrap on a case-sensitive filesystem, such as the production Linux image;
- regenerate optimized metadata in the release build, then run class-loading tests against that exact artifact;
- if using classmap-authoritative mode, test any framework or tool that creates classes at runtime.

If a class is missing, first inspect its fully qualified name, source declaration, generated mapping, and file casing. Then check whether the loader was registered before the class was needed and whether another callback threw or took precedence. Re-running `dump-autoload` can fix stale generated metadata, but it cannot repair a wrong namespace, a missing file, or an API incompatibility.

## Common Mistakes

- Expecting PHP to autoload functions or constants from a class file.
- Assuming `vendor/autoload.php` installs dependencies; it only registers the generated loader and includes configured eager files.
- Registering a callback that throws whenever a class is not its responsibility, preventing later callbacks from trying.
- Converting arbitrary user-controlled class names to paths with no namespace or path policy.
- Assuming an optimized or authoritative map will automatically notice files added after build time.
- Treating `files` autoloading as lazy, even though those files load during autoloader bootstrap.
- Copying generated Composer autoload files from a different source revision or build artifact.
- Enabling APCu or authoritative mapping without testing in the same PHP SAPI and release environment used in production.

## Exercises

1. Run the two-file SPL example. Add a second callback registered after the first, and observe that it is attempted only when the first callback leaves the name undefined.
2. Change the `Report` file path’s letter casing and run the test on a case-sensitive filesystem. Record why a local workstation might not reveal the problem.
3. Build a small Composer project with PSR-4 and one `files` entry. Confirm that constructing a class triggers class lookup, while the function file is included at autoloader bootstrap.
4. Generate an optimized class map, add a class, and compare ordinary optimization with authoritative mode before and after regenerating the loader.
5. Measure the memory and startup behavior of an application with its normal loader, optimized class map, and APCu enabled. Use production-like code and report the workload and environment.

## Review Questions

1. What does PHP do when it encounters a class-like name that has not yet been defined?
2. Why does autoloading work for classes and interfaces but not functions or constants?
3. What is the difference between `spl_autoload_register()` and an explicit `require`?
4. How do Composer’s PSR-4, classmap, and `files` strategies differ at runtime?
5. What happens if an autoloader throws before a later callback can handle the requested class?
6. How does an authoritative class map change fallback behavior, and what code can it break?
7. What does APCu caching add, and which operational resource does it consume?

## Summary

PHP calls registered autoloaders when code needs a class, interface, trait, or enum that is not yet defined. `spl_autoload_register()` maintains an ordered callback queue; a callback that cannot handle a name should return so later callbacks can try. Functions and constants need explicit file inclusion or Composer’s eager `files` mechanism.

Composer generates `vendor/autoload.php` from project and dependency metadata. The application includes it during bootstrap, and PHP then invokes the generated loader on demand. PSR-4 is the common namespace mapping strategy, PSR-0 remains for legacy code, class maps provide direct name-to-file lookup, and `files` includes code eagerly. Optimized maps can reduce lookup work; authoritative mode removes filesystem fallback, and APCu caches lookup results. Regenerate loader metadata from the exact release tree and test any stricter mode against runtime-generated classes.

## References

- [PHP Manual: Autoloading Classes](https://www.php.net/manual/en/language.oop5.autoload.php)
- [PHP Manual: `spl_autoload_register()`](https://www.php.net/manual/en/function.spl-autoload-register.php)
- [PHP Manual: `class_exists()`](https://www.php.net/manual/en/function.class-exists.php)
- [Composer: Autoloading](https://getcomposer.org/doc/01-basic-usage.md#autoloading)
- [Composer: Autoloader optimization](https://getcomposer.org/doc/articles/autoloader-optimization.md)
- [Composer schema: Autoloading](https://getcomposer.org/doc/04-schema.md#autoload)
- [Chapter 92 — Composer](092-composer.md)
- [Chapter 94 — `composer.json`](094-composer-json.md)
- [Chapter 98 — PSR-4](098-psr-4.md)
