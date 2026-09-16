---
book: The Complete Modern PHP Engineering Book
volume: 4
volume_title: PHP UNDER THE HOOD
chapter: 58
title: Extensions
slug: extensions
status: complete
summary: ../../_ai/chapter-summaries/058-extensions-summary.md
---

# Chapter 58 — Extensions

## Why This Matters

The PHP language is only part of the runtime a production application uses. PDO, cURL, OpenSSL, mbstring, JSON, database drivers, OPcache, and framework-adjacent libraries cross the extension boundary. An extension can add functions, classes, constants, stream wrappers, INI settings, hooks, serializers, and native integrations. It can also change startup requirements, memory behavior, portability, and security posture.

When `json_encode()` is fast, a database driver returns a resource-like object, or `password_hash()` delegates to a cryptographic implementation, userland PHP is not doing all the work. Understanding the boundary helps explain why two machines with the same application code can behave differently and why a PHP upgrade may require rebuilding or replacing a binary extension.

## Mental Model

```text
PHP source
    → compiled/userland call
    → internal function or class registered by an extension
    → Zend Engine API and native library
    → operating system / network / database / hardware
```

An extension may be built into PHP or loaded as a shared module. The module is loaded by a specific PHP binary and SAPI configuration. CLI and PHP-FPM can therefore have different extension sets and INI files. “It works in the terminal” is not evidence that the web worker loaded the same module.

The extension API is not the same thing as the public PHP language API. A function's documented behavior is the application contract; a `zend_*` struct layout or macro expansion is an implementation/build contract. Internal APIs change across major PHP versions and can require source changes when moving from PHP 5 to PHP 7/8.

## Core Concept: Module and Request Lifecycles

An extension participates in process and request lifecycles conceptually like this:

```text
module load
    → MINIT: process/module initialization
    → RINIT: request initialization
    → userland code runs
    → RSHUTDOWN: request cleanup
    → MSHUTDOWN: process/module shutdown
```

Not every extension needs every hook. The exact lifecycle differs by SAPI, CLI invocation, persistent worker behavior, and whether the module is statically compiled or dynamically loaded. The important operational distinction is process-lifetime state versus request-lifetime state. A module global that is not reset at request boundaries can leak data between requests or create concurrency bugs in a threaded build.

PHP-FPM normally uses multiple processes, so ordinary process globals are not shared across workers. A ZTS build has additional thread-safety constraints, but ZTS does not make arbitrary native state safe. Treat module globals and native-library handles as explicitly owned resources.

## What PHP Does

Use the running binary to inspect the actual environment:

```text
php -v
php --ini
php -m
php --ri opcache
php -r 'var_dump(PHP_SAPI, PHP_VERSION, extension_loaded("curl"));'
```

For a web SAPI, inspect a protected diagnostics path or startup logs rather than assuming CLI configuration. `phpinfo()` is useful during controlled diagnosis but can disclose paths, versions, environment details, and configuration; never publish it unprotected.

Userland can inspect an extension in a limited way:

```php
<?php

declare(strict_types=1);

if (!extension_loaded('mbstring')) {
    throw new RuntimeException('mbstring is required');
}

$extension = new ReflectionExtension('mbstring');
printf("%s %s\n", $extension->getName(), $extension->getVersion() ?? 'built-in');
```

Availability checks are not a substitute for a deployment requirement. If an application cannot operate safely without an extension, fail during startup or deployment validation rather than taking a degraded path halfway through a request.

## Minimal Example: An Internal Function

An internal function has a native implementation and an argument description (arginfo). A simplified PHP 7/8-style illustration is:

```c
PHP_FUNCTION(example_add)
{
    zend_long left;
    zend_long right;

    ZEND_PARSE_PARAMETERS_START(2, 2)
        Z_PARAM_LONG(left)
        Z_PARAM_LONG(right)
    ZEND_PARSE_PARAMETERS_END();

    RETURN_LONG(left + right);
}

ZEND_BEGIN_ARG_WITH_RETURN_TYPE_INFO_EX(arginfo_example_add, 0, 2, IS_LONG, 0)
    ZEND_ARG_TYPE_INFO(0, left, IS_LONG, 0)
    ZEND_ARG_TYPE_INFO(0, right, IS_LONG, 0)
ZEND_END_ARG_INFO()

static const zend_function_entry example_functions[] = {
    PHP_FE(example_add, arginfo_example_add)
    PHP_FE_END
};
```

This is intentionally not a complete buildable extension. It shows three boundaries: parameter parsing, a native operation, and registration metadata. Macro signatures and generated build files depend on the PHP version and extension skeleton. Native code must validate integer overflow, ownership, failure returns, and cleanup; `left + right` is only a teaching example.

The function table makes the native symbol visible to the engine. The engine then creates the callable function entry used by compiled userland code. At runtime the VM dispatches the call through an internal-function handler rather than executing a userland function body.

## How It Works: Registration and ABI

At module startup, the extension registers functions, classes, constants, handlers, and INI entries with the engine. The engine stores callable metadata in internal tables. When source calls the function, compiled opcodes identify the callable and the VM enters native code through the internal function interface.

The PHP extension API includes ABI-sensitive structures and macros. A module compiled for one PHP major/minor line is not automatically safe for another. Distribution packages normally build extensions against the exact PHP ABI they ship. Tools such as PECL automate common builds, but they do not remove the need to verify compatibility, compiler flags, linked libraries, and runtime configuration.

PHP 7 replaced many PHP 5 extension APIs, and PHP 8 continued tightening signatures and engine contracts. Modern extension code should provide accurate arginfo, use current parameter-parsing macros, check nullable and union behavior explicitly, and test against every supported PHP minor release. Do not copy a PHP 5 `zval` manipulation example into a PHP 8 extension and expect it to be valid.

## Practical Example: Configuration Is Part of the Artifact

An application using cURL and OpenSSL has at least these deployment questions:

1. Is the extension present in the CLI image and the FPM image?
2. Is the linked library version supported by the extension build?
3. Does the web SAPI load the intended INI file?
4. Are CA certificates present in the container?
5. Do staging and production use the same architecture and build flags?
6. Does the extension allocate native buffers whose limits are outside the PHP code's assumptions?

A startup check can make the contract visible:

```php
<?php

declare(strict_types=1);

$required = ['json', 'openssl', 'pdo'];

foreach ($required as $name) {
    if (!extension_loaded($name)) {
        throw new LogicException("Required extension is missing: {$name}");
    }
}
```

Some extensions are built into a particular PHP distribution and may report a version differently from a shared module. Check documented capabilities, not only a version string, when the behavior depends on a feature.

## Production Example: Extension Failure Boundaries

An extension wraps a native boundary where failures have different forms: a function can return `false`, throw a `Throwable`, emit a warning, set an error code, or terminate on an unrecoverable allocation failure. The application adapter should normalize documented failures into a domain-level result or exception while retaining the original context.

```php
final class ImageDecoder
{
    public function decode(string $bytes): Image
    {
        $image = imagecreatefromstring($bytes);
        if ($image === false) {
            throw new InvalidArgumentException('Input is not a supported image');
        }

        try {
            return $this->convert($image);
        } finally {
            imagedestroy($image);
        }
    }

    private function convert(GdImage $image): Image
    {
        // Domain conversion omitted.
        return new Image();
    }
}
```

The exact function and return types depend on the GD version; the example's lesson is ownership and cleanup at an extension boundary. In production, cap input dimensions and bytes before decoding because decompressed images can be much larger than their upload size.

## Bad Example

```php
if (PHP_SAPI === 'cli') {
    $config = require __DIR__ . '/cli-config.php';
} else {
    $config = require __DIR__ . '/web-config.php';
}

// Assume all extension behavior is identical.
```

SAPI selection does not establish identical module lists, INI scan directories, linked libraries, environment variables, or architecture. A deployment that tests only CLI can miss a missing FPM driver or a different OPcache setting.

## Better Example

Make environment verification explicit and observable:

```php
final class RuntimeRequirements
{
    /** @param list<string> $extensions */
    public static function check(array $extensions): void
    {
        $missing = array_values(array_filter(
            $extensions,
            static fn (string $name): bool => !extension_loaded($name),
        ));

        if ($missing !== []) {
            throw new LogicException('Missing PHP extensions: ' . implode(', ', $missing));
        }
    }
}
```

Run this check in the same image and SAPI that serves traffic. Keep a separate compatibility matrix for PHP version, extension version, OS libraries, and architecture. Composer's `ext-*` platform requirements can fail installation early, but runtime smoke tests still matter for native behavior.

## Performance

An extension can reduce CPU cost by moving tight loops or system calls into native code, but the boundary is not free. Argument conversion, allocation, copying, encoding, and external I/O may dominate. A userland loop over a small dataset can be faster to maintain and sufficiently fast; a native call that copies a 2 GB buffer can still be memory-bound.

Benchmark the complete operation, including input conversion and output ownership. Record PHP version, extension version, library version, SAPI, CPU architecture, and configuration. Do not compare a debug build with a distribution release or infer web latency from a CLI microbenchmark.

## Security

Extensions expand the trusted computing base. A memory-safety bug in native code can compromise the process more severely than a userland exception. Minimize installed modules, use maintained distribution packages, track CVEs for both the extension and linked libraries, and rebuild rather than copying `.so` files across incompatible images.

Validate untrusted lengths and dimensions before passing data to native libraries. Keep network and filesystem permissions narrow. Do not enable an extension because a library “might need it” without understanding the functions and attack surface it adds.

## Testing and Verification

Use multiple layers:

- Unit-test the userland adapter with fakes for domain behavior.
- Run extension-backed integration tests against the actual module and linked service/library.
- Run a startup matrix for CLI and FPM, including required INI settings.
- Test false returns, warnings, exceptions, malformed input, oversized input, and timeouts.
- Run memory and repeated-operation tests for native handles and long-lived workers.
- Test each supported PHP minor version and architecture in CI where the extension ABI or behavior differs.

Useful commands include `php --ini`, `php -m`, `php --ri extension-name`, and `php -r 'var_dump(extension_loaded("extension-name"));'`. Preserve the command output as build metadata rather than depending on a developer's local installation.

## Common Mistakes

- Assuming CLI and FPM load the same extensions.
- Treating a PHP extension as a portable application-level dependency without an ABI/build contract.
- Omitting arginfo or using stale PHP 5 extension APIs in modern code.
- Ignoring native memory and cleanup because userland variables went out of scope.
- Passing unbounded input to image, archive, compression, or parser extensions.
- Exposing `phpinfo()` or extension diagnostics publicly.
- Testing only successful return values.

## Senior Engineer Thinking

For every extension, identify the public contract, lifecycle, ownership rules, failure modes, ABI/build requirements, and operational evidence. Then decide whether the capability belongs in the base image, a separate worker image, or an external service. Native code is often the right tool for a narrow problem, but it is not free abstraction: it moves complexity into builds, upgrades, memory, and security review.

## Exercises

1. Compare `php --ini`, `php -m`, and a protected FPM diagnostics output. Document every difference.
2. Write an adapter around an extension function that can return `false` or throw. Test both paths and preserve context.
3. Build a CI compatibility matrix for two PHP minor versions and one extension with a native library dependency.
4. Trace a userland call to an internal function using `ReflectionFunction` and the PHP source's function table. Label documented behavior versus implementation detail.

## Review Questions

1. What does an extension add to the PHP runtime besides functions?
2. Why are MINIT/RINIT and RSHUTDOWN/MSHUTDOWN useful lifecycle concepts?
3. Why can CLI and FPM disagree about extension availability?
4. What role does arginfo play in a modern internal function?
5. Why is a shared-object file not automatically portable between PHP installations?
6. Which tests are needed at the extension boundary that a pure unit test cannot provide?

## Summary

Extensions connect the Zend Engine to native code, libraries, and system services. They register callable metadata and participate in module and request lifecycles, but their ABI, memory ownership, failure behavior, and security surface are build- and version-sensitive. Inspect the actual SAPI environment, validate requirements at startup, wrap native failures deliberately, and test the deployed extension rather than only a userland substitute.

## Official References

- [PHP Internals Book: extensions](https://www.phpinternalsbook.com/php7/extensions_design/)
- [PHP manual: extension loading](https://www.php.net/manual/en/extensions.php)
- [PHP manual: building PHP from source](https://www.php.net/manual/en/install.unix.php)
- [PHP source: extension directory](https://github.com/php/php-src/tree/master/ext)
- [PHP source: Zend API headers](https://github.com/php/php-src/tree/master/Zend)
- [Composer platform packages](https://getcomposer.org/doc/articles/composer-platform-dependencies.md)
