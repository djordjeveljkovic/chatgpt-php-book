---
book: The Complete Modern PHP Engineering Book
volume: 1
volume_title: THE PHP MENTAL MODEL
chapter: 3
title: PHP as a Language vs PHP as a Runtime
slug: php-as-a-language-vs-php-as-a-runtime
status: complete
summary: ../../_ai/chapter-summaries/003-php-as-a-language-vs-php-as-a-runtime-summary.md
---

# Chapter 3 — PHP as a Language vs PHP as a Runtime

## Why This Matters

“PHP” is used to name several different things at once. It can mean the language rules in a source file, the `php` executable, the Zend Engine inside that executable, an extension such as PDO, a PHP-FPM worker, or the complete environment containing a web server, configuration, operating system, and database.

That shorthand is convenient until something goes wrong. A team says that “PHP changed the value,” when the actual cause is a coercion rule. Someone says that “PHP cannot connect to MySQL,” when the CLI has a driver that the FPM workers do not. A developer concludes that “the variable disappeared,” when the request ended and a new process began. Each explanation points at PHP, but each points at a different layer.

The distinction matters because each layer has a different kind of solution. A language problem is addressed with code and an understanding of semantics. A missing extension is addressed by building or loading the capability. A configuration mismatch is addressed by changing deployment configuration. A database timeout is addressed at the network, database, or failure-handling boundary. Changing a loop cannot repair a missing extension, and changing `php.ini` cannot make an invalid expression valid PHP.

This chapter builds a vocabulary for making those distinctions deliberately.

## Mental Model

Treat a running PHP program as a stack of contracts:

```text
PHP source code
    ↓
Language semantics
    ↓
PHP implementation and runtime
    ↓
Extensions and SAPI
    ↓
Configuration
    ↓
Process and operating environment
    ↓
External services
```

The layers interact, but they should not be collapsed into one concept.

The language semantics answer questions such as:

- What does `1 + 2` evaluate to?
- What does a function declaration require from its arguments?
- What does `throw` do to the current call stack?
- Which statements are valid syntax?

The implementation and runtime answer questions such as:

- How are values represented in memory?
- How is source transformed into executable instructions?
- How are function calls, allocations, errors, and output performed?
- What memory and CPU costs result from the implementation?

Extensions and the Server Application Programming Interface (SAPI) answer questions such as:

- Is `PDO` available?
- Which database drivers can it use?
- Is the process running through CLI, FPM, CGI, or another interface?
- Which input and output channels are connected to the process?

Configuration answers questions such as:

- What is the `memory_limit`?
- Are errors displayed or logged?
- Which `php.ini` file was loaded?
- Is an extension enabled?

The environment answers questions such as:

- What files exist and which user can access them?
- What environment variables and network routes are present?
- Is the process short-lived or long-running?
- Is the database reachable and healthy?

This is a reasoning model, not a claim that the executable has five perfectly isolated modules. The boundaries are useful because they identify ownership of behavior.

## Core Concept

### Language semantics are the meaning of PHP code

The language is the part that can be discussed without choosing a web server. It includes syntax, expressions, variables, types, functions, objects, control flow, exceptions, and other constructs defined by a PHP version.

For example:

```php
<?php

declare(strict_types=1);

function add(int $left, int $right): int
{
    return $left + $right;
}

echo add(2, 3), PHP_EOL;
```

At the language level, the function accepts two integer arguments, adds them, returns an integer, and the caller evaluates `echo` with the returned value. These facts belong to the program's meaning. They are not properties of HTTP, PHP-FPM, or MySQL.

The language also has version boundaries. A syntax feature introduced in PHP 8 cannot be assumed to parse on PHP 7.4. A behavior that was deprecated in one version may be removed in another. “Valid PHP” therefore always means “valid for a particular PHP version and execution mode.” The version is part of the language contract a project supports.

### Runtime behavior is how the meaning is carried out

The runtime turns source into observable behavior. Conceptually, the path is:

```text
Source text
    ↓
Lexing and parsing
    ↓
Compiled representation
    ↓
Zend VM execution
    ↓
Output and side effects
```

The exact internal representation and execution strategy are implementation details. The important engineering consequence is that the same language operation has runtime costs. Assigning a large array, calling a function in a tight loop, allocating many temporary strings, or retaining objects in a worker all have consequences beyond the source-level spelling.

OPcache can keep compiled script data available for reuse, so a later request may avoid repeating some compilation work. That optimization does not change what a correct expression means. It changes the cost of reaching execution and introduces operational concerns such as cache invalidation and deployment restarts.

The Zend Engine is the core runtime used by the standard PHP implementation. It supplies the machinery for executing PHP code and for managing values, calls, errors, memory, and internal APIs. Later volumes will examine zvals, HashTables, opcodes, the Zend VM, OPcache, and process lifecycles. This chapter only establishes the boundary: the language describes the operation; the engine realizes it.

### Extensions add capabilities at the runtime boundary

Core PHP does not contain every capability an application may need. Extensions can add functions, classes, constants, stream wrappers, serializers, database interfaces, cryptographic operations, image processing, and other facilities. Some extensions are bundled or commonly enabled; others must be installed and loaded explicitly.

Consider this code:

```php
<?php

if (!extension_loaded('pdo')) {
    throw new RuntimeException('PDO is required');
}

$connection = new PDO($dsn, $username, $password);
```

The `if` statement and exception are language features. `extension_loaded()` and `PDO` are runtime-provided APIs. Whether a particular PDO driver can connect to a particular database also depends on the driver, credentials, network, and database server. One line of source can therefore cross several boundaries.

An extension is not the same thing as a library written in PHP. A Composer package can provide userland classes and functions while relying on an extension underneath. For example, a database abstraction may call PDO, and PDO may require a database-specific driver. Installing the package does not necessarily install the extension or make the database reachable.

This distinction explains a common deployment failure:

```text
Application code
    ↓ calls
PDO API
    ↓ requires
PDO driver extension
    ↓ speaks to
Database server
```

The application can be syntactically correct and still fail before a query is sent because the driver is absent.

### Configuration changes runtime conditions, not language meaning

PHP configuration is a set of inputs to the runtime. It can come from `php.ini`, additional configuration files, SAPI-specific settings, command-line options, or runtime calls where the directive permits them. The changeability of a directive matters: some settings can be changed per script, while others are system or startup settings.

For example:

```php
<?php

echo 'SAPI: ', PHP_SAPI, PHP_EOL;
echo 'Loaded ini: ', php_ini_loaded_file() ?: '(none)', PHP_EOL;
echo 'Memory limit: ', ini_get('memory_limit'), PHP_EOL;
```

This code has stable language semantics, but its output is environment-specific. `memory_limit` can change how much work succeeds. `display_errors` can change whether an error is sent to the output stream. `date.timezone` can change how date operations are interpreted when no explicit timezone is supplied. None of these settings changes the grammar of `if`, the meaning of addition, or the visibility rules of a class.

A useful rule is:

> Configuration can change the conditions under which a valid program runs; it does not make arbitrary invalid PHP valid.

There are exceptions in the broad historical sense—settings such as `short_open_tag` affect whether a particular tag form is recognized—but that is precisely why configuration must be treated as part of the deployment contract. Avoid source forms whose validity depends on optional configuration. The normal `<?php` tag and `<?=` short echo tag are the portable choices.

### The environment supplies the world outside PHP

The operating system and surrounding services provide resources PHP cannot guarantee. Files, sockets, DNS, process signals, clocks, permissions, environment variables, and external services belong to this layer.

```php
<?php

$path = __DIR__ . '/var/report.json';
$contents = @file_get_contents($path);

if ($contents === false) {
    throw new RuntimeException("Could not read {$path}");
}
```

The call and return-value check are PHP behavior. Whether the file exists, whether the worker's user can read it, whether the filesystem is mounted, and whether the disk is healthy are environmental facts. A correct program must handle both sides of that boundary.

Environment variables are another example. `getenv('APP_ENV')` reads a process environment value, but PHP does not automatically decide that every environment variable is valid application configuration. A framework or bootstrap layer may parse, validate, and map it into typed application settings. Senior code treats that mapping as an explicit boundary rather than scattering calls to `getenv()` through business logic.

## How It Works

When a PHP program starts, the runtime first establishes an execution context. The SAPI identifies how PHP was invoked. Startup configuration is loaded, extensions are initialized, and process-level resources become available. The runtime then receives source code, parses and compiles it, and executes the resulting instructions.

For a CLI command:

```text
Shell
  → PHP CLI SAPI
  → startup configuration and extensions
  → script execution
  → stdout/stderr and exit code
```

For a typical web request:

```text
Client
  → web server or reverse proxy
  → FastCGI connection
  → PHP-FPM worker
  → startup/request context
  → script execution
  → response bytes and status
```

The source file can be identical in both cases. The process inputs, output consumer, configuration, and lifetime are not.

A runtime can reject a program before executing its body. A parse error belongs to the source-to-program boundary. A missing class or undefined function may belong to extension loading or autoloading. A `TypeError` may occur while invoking a valid function declaration with invalid arguments. A connection timeout may occur after PHP has correctly executed the call and handed work to an external system. Categorizing the failure by stage is more useful than calling all of them “PHP errors.”

## What PHP Does

PHP supplies the language and the standard runtime APIs used by applications. It can run scripts from the command line, process web requests, execute queue workers, transform files, communicate with databases, and talk to network services. HTML is one output format, not the definition of the language.

The same domain operation should usually be independent of its entry point:

```php
<?php

function canReserve(string $requestedCourt, string $occupiedCourt): bool
{
    return $requestedCourt !== $occupiedCourt;
}
```

The function is language-level application code. An HTTP controller may obtain `$requestedCourt` from a request. A CLI command may obtain it from `$argv`. A queue worker may obtain it from a message. The adapters differ; the domain rule need not.

This separation gives the team a more precise answer to “where should this behavior live?” Input decoding, authentication, and response formatting belong near the entry point. Business rules belong in code that can run under multiple entry points. Runtime and environment checks belong in bootstrap or infrastructure boundaries.

## What Zend Does

At a high level, the Zend Engine manages the execution of the compiled PHP program. It represents values, resolves variables and functions, invokes userland and internal functions, handles exceptions, and coordinates memory management. Extensions use runtime interfaces to register capabilities that PHP code can call.

The engine's behavior is not the same as the language's public contract. A zval layout, a bucket arrangement in a HashTable, or a particular opcode sequence is an implementation detail. It matters when explaining memory, speed, or debugging, but application code should not depend on private layout details.

This distinction prevents two opposite mistakes:

1. Treating implementation details as guarantees. A measurement on one PHP build is not automatically a language rule.
2. Treating runtime costs as imaginary because the source looks simple. A one-line array operation can allocate, copy, hash, or invoke user code.

The right level of explanation depends on the question. To understand why a function returns a value, language semantics may be enough. To understand why a worker's memory grows, runtime representation and process lifetime become necessary.

## Minimal Example

The following example separates a language rule from its execution context:

```php
<?php

declare(strict_types=1);

function greet(string $name): string
{
    return "Hello, {$name}";
}

echo greet('Ada'), PHP_EOL;
```

The declaration, interpolation, function call, return, and `echo` are language constructs. The newline is emitted to the process output stream. In CLI, that stream is normally the terminal or a redirected file. In a web SAPI, the bytes may become part of an HTTP response. The source does not itself specify that a browser is present.

## Practical Example

Suppose an application reports that a database feature works from a developer shell but fails in production. Inspect the layers explicitly:

```php
<?php

printf("PHP %s\n", PHP_VERSION);
printf("SAPI %s\n", PHP_SAPI);
printf("PDO %s\n", extension_loaded('pdo') ? 'loaded' : 'missing');
printf("MySQL driver %s\n", extension_loaded('pdo_mysql') ? 'loaded' : 'missing');
printf("INI %s\n", php_ini_loaded_file() ?: '(none)');
```

Run an equivalent diagnostic in the same deployment context as the failing code. `php -m` from a shell describes the CLI binary; it does not prove that FPM workers loaded the same modules. A safe diagnostic should emit only non-sensitive facts, be access-controlled, and be removed or disabled after investigation.

The diagnosis may then be precise:

- the language code is valid;
- the production FPM SAPI uses a different configuration;
- `pdo_mysql` is absent from that runtime;
- or the driver is present but the database host is unreachable.

Each result leads to a different fix.

## Production Example

A reliable deployment treats the runtime as an artifact with a contract. The contract should specify at least:

- the supported PHP version and architecture;
- the SAPI used by web requests and workers;
- required extensions and their drivers;
- startup configuration and important limits;
- environment variables and their validation rules;
- filesystem, network, and process permissions;
- external service dependencies.

For example, a queue worker may use the same application code as an HTTP request but have a much longer process lifetime. It must not assume that request cleanup will occur after every message, and it may need explicit handling for stale connections, accumulated memory, signals, and retries. Those concerns are runtime and environment concerns, even though the worker is written in PHP.

## Bad Example

This code hides a deployment problem:

```php
<?php

if (function_exists('imagecreatetruecolor')) {
    return renderChart();
}

return renderPlaceholder();
```

If chart generation is a required business capability, silently rendering a placeholder creates a false success. The request may return a page that looks valid while losing an important artifact. The missing GD capability is an operational defect, not an optional product decision.

Another bad assumption is:

```php
<?php

// The CLI works, so the web application has the same PHP setup.
```

CLI and FPM can have different binaries, `php.ini` files, enabled extensions, users, working directories, limits, and environment variables.

## Better Example

Validate required capabilities during bootstrap and fail with an actionable message:

```php
<?php

declare(strict_types=1);

$requiredExtensions = ['pdo', 'pdo_mysql', 'mbstring'];

foreach ($requiredExtensions as $extension) {
    if (!extension_loaded($extension)) {
        throw new RuntimeException(
            "Required PHP extension is missing: {$extension}"
        );
    }
}
```

The application can still choose an intentional fallback when the feature is genuinely optional. The important point is that the choice is explicit and observable. Startup validation turns a late request failure into a deployment failure that can be detected before traffic is served.

## Edge Cases

### The same version is not necessarily the same runtime

`PHP_VERSION` alone is not a complete description. Two installations can report the same version while differing in SAPI, loaded extensions, build options, configuration, operating system, CPU architecture, and linked client libraries.

### A configuration value can be unavailable or misleading

`ini_get()` returns a string representation and may return `false` for an unknown setting. Some directives cannot be changed at runtime. Code that needs a numeric limit should parse and validate it rather than assuming every configuration value is already a typed integer.

### A loaded extension does not guarantee a healthy service

`extension_loaded('pdo_mysql')` proves that the driver is available. It does not prove that DNS works, credentials are correct, the database accepts connections, or a query will finish before its timeout.

### Process lifetime changes what state means

A local variable normally belongs to one execution of a script. A worker process can retain global state, static state, object graphs, caches, and open resources across many jobs. The language rule for variable scope has not changed; the environment has changed the lifetime during which that state remains reachable.

### Output is not automatically an HTTP response

`echo` writes output through the active SAPI. It does not, by itself, define a status code, content type, security headers, or response body boundaries. Those are responsibilities of the web boundary and its protocol handling.

## Performance

Performance questions should be assigned to the correct layer.

Language-level complexity still matters. A linear scan through a collection remains an O(n) algorithm regardless of whether the script runs in CLI or FPM. Choosing a hash map, an index, or a streaming approach can change the algorithmic cost.

Runtime behavior adds constant factors and resource costs. Value representation, allocation, reference counting, copying, function calls, compilation, opcode caching, and garbage collection can affect CPU and memory. These are reasons to understand the runtime, not reasons to replace measured analysis with folklore.

The environment often dominates. A database round trip or remote API call can cost much more than the PHP instructions surrounding it. A saturated FPM pool can make a fast script appear slow because requests are waiting for a worker. A memory limit can turn a theoretically acceptable operation into an out-of-memory failure.

Measure in the production-like SAPI, configuration, workload, and dependency conditions. A CLI microbenchmark is evidence about that CLI setup; it is not automatically evidence about an FPM request or a long-running worker.

## Security

Runtime configuration and extensions are part of the security boundary. Keep production diagnostics such as `phpinfo()` and detailed configuration dumps restricted; they can disclose paths, versions, loaded modules, and environment details. Do not treat settings such as `open_basedir` or `disable_functions` as a complete application security model. Enforce authorization, validate input, isolate processes, and apply operating-system permissions independently.

Treat extensions and native dependencies as deployed software. Pin and review the image or package that supplies them, keep web and worker runtimes aligned, and remove capabilities that are not required. Protect PHP-FPM from untrusted FastCGI clients: a client that can directly control a FastCGI request can influence how PHP processes it.

Environment variables also need care. They are convenient for configuration, but they can leak through debug output, process inspection, crash reports, or accidental logging. Load secrets at a controlled boundary, validate them, and avoid carrying them into places that do not need them.

## Database Interaction

The database boundary demonstrates all five distinctions at once:

- PHP language semantics govern the code that builds and executes a call.
- The PHP runtime provides the object and function machinery.
- PDO and its database driver provide the client API.
- Configuration supplies driver settings, timeouts, and credentials.
- The network and database server determine whether the operation succeeds.

Even SQL has its own language semantics and runtime. A PHP expression can be correct while a SQL statement is invalid. A valid SQL statement can still block on a lock or fail because the server is unavailable. Keep the PHP exception handling and the database transaction policy explicit rather than assuming that calling a method means the side effect has committed.

## Concurrency

The PHP language does not promise that separate requests share ordinary local variables. Whether state is shared depends on the process model and the resource used to store it. Separate FPM workers generally have separate memory, while a shared database, cache, filesystem, or message broker can be observed by many processes.

This is why an in-memory “check then insert” sequence cannot by itself enforce a cross-request invariant. Two workers can execute the check concurrently. The database or another coordination mechanism must enforce the rule when the requirement spans processes.

The same reasoning applies to workers. A long-running process can share its own retained state across jobs even when separate workers do not share memory with one another. Design cleanup, idempotency, locking, and restart behavior for the actual worker topology.

## Testing

Test each contract at the layer where it lives:

- Unit-test language-level application rules without requiring a SAPI or external service.
- Test bootstrap validation with required and missing extensions/configuration.
- Run the suite across every supported PHP version.
- Run integration tests with the same important extensions and drivers as production.
- Exercise web behavior through the web SAPI and worker behavior in a long-lived process when those lifecycles matter.
- Test dependency failures such as connection refusal, timeout, permission denial, and malformed responses.

When a test fails only in CI or only in production, compare runtime facts before changing application logic: PHP version, SAPI, loaded modules, loaded INI files, working directory, user, environment, and dependency endpoints.

## Common Mistakes

- Calling every failure “a PHP problem” without identifying the failing layer.
- Assuming the PHP language includes every extension or Composer package an application uses.
- Checking extensions in CLI and assuming FPM has the same modules.
- Reading environment variables throughout domain code instead of validating configuration once.
- Treating `ini_set()` as if every directive were changeable at runtime.
- Depending on optional syntax or behavior controlled by deployment configuration.
- Measuring a local CLI script and using the result to make claims about FPM or workers.
- Relying on in-process state to coordinate independent requests.
- Exposing full runtime diagnostics in a production endpoint.
- Treating a loaded driver as proof that the external database is healthy.

## Senior Engineer Thinking

When debugging, write down the claim in a form that identifies the layer:

```text
Observed: the report endpoint fails
Language: the source parses and the function call is valid
Runtime: the FPM worker loaded the application
Extension: the database driver is present
Configuration: the timeout and credentials are configured
Environment: DNS and network access work
External service: the database accepts the query
```

Do not fill in those statements from memory. Inspect the failing execution context. Compare a successful context with a failing one. Then fix the narrowest layer that owns the defect.

This approach also improves design reviews. Ask whether a proposed guarantee is a language guarantee, a runtime property, a configuration assumption, or an external invariant. Ask what happens when the process restarts, the extension is missing, the setting differs, or the dependency is slow. Those questions expose hidden assumptions before production does.

The most useful mental model is not “PHP is interpreted” or “PHP is compiled.” PHP source is parsed and compiled into an executable representation, then executed by a runtime; optional caching and optimization can change the path and cost. The useful question is always: which contract is relevant to this decision, and what evidence supports it?

## Exercises

1. Run `php --ini`, `php -m`, and a short script printing `PHP_VERSION` and `PHP_SAPI`. Record which facts describe the CLI runtime and which facts would still need to be checked for web requests.
2. Choose one application capability, such as image generation or database access. Draw its path from PHP source through the extension and configuration to the external service. Mark one failure that belongs to each layer.
3. Take a function used by an HTTP controller and identify the parts that depend on HTTP. Refactor the explanation, not necessarily the code, so the core rule could also be called by a CLI command and a queue worker.
4. Design a startup check for a required extension and configuration value. Decide what should fail fast, what should be optional, and what information is safe to include in the failure message.

## Review Questions

1. What is the difference between a language semantic rule and a runtime implementation detail?
2. Why can the same PHP source behave differently under CLI and PHP-FPM?
3. What does a loaded extension prove, and what does it not prove?
4. Why should environment variables be mapped into validated application configuration?
5. Why is a database invariant not usually enforceable by PHP memory alone?
6. Which performance claims require measurement in the target SAPI and process model?
7. What facts would you compare first when code works in CLI but fails through the web application?

## Summary

PHP is both a language and a running system, but those descriptions answer different questions. Language semantics define the meaning of source code. The implementation and Zend Engine parse, compile, execute, and manage that code. Extensions add runtime capabilities. Configuration changes the conditions in which the runtime operates. The operating environment supplies processes, files, network access, permissions, and external services.

Senior engineers attribute behavior to the narrowest layer that controls it. They separate a valid PHP program from the runtime that executes it, a loaded extension from a healthy dependency, and a CLI installation from an FPM deployment. They test and measure under the real process model, validate runtime requirements at startup, and enforce cross-process invariants at the appropriate shared boundary.

Once these distinctions are clear, later topics become easier to place. Variables and arrays can be explained as language features with runtime representations. OPcache and PHP-FPM can be explained as runtime and process mechanisms. Configuration, databases, concurrency, security, and production failures can be discussed without attributing every outcome to “PHP” as one undifferentiated object.

## Sources and Further Reading

- [PHP Manual: What is PHP and what can it do?](https://www.php.net/whatisphp)
- [PHP Manual: Language Reference](https://www.php.net/manual/en/langref.php)
- [PHP Manual: Internal functions](https://www.php.net/manual/en/functions.internal.php)
- [PHP Manual: Runtime Configuration](https://www.php.net/manual/en/configuration.php)
- [PHP Manual: Core `php.ini` directives](https://www.php.net/manual/en/ini.core.php)
- [PHP Manual: Command line usage](https://www.php.net/manual/en/features.commandline.usage.php)
- [PHP Manual: FastCGI Process Manager (FPM)](https://www.php.net/manual/en/install.fpm.php)
