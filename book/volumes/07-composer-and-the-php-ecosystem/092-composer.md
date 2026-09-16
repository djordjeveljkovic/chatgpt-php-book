---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 92
title: Composer
slug: composer
status: complete
summary: ../../_ai/chapter-summaries/092-composer-summary.md
---

# Chapter 92 — Composer

## Why This Matters

A PHP application rarely consists only of files its team wrote. It may rely on a logging library, a database client, a test runner, a framework, or packages maintained elsewhere. Without a shared dependency workflow, each developer can end up with a different set of library files, and a production host may not have the code that was present on a developer's laptop.

Composer gives PHP projects a common way to describe packages, acquire them, prepare the project tree, and make classes available to the application. This puts a repeatable boundary around third-party code. It also makes updates, security review, test tools, and deployment part of an explicit project lifecycle rather than a set of manual downloads.

Composer is a tool in the development and build environment. It is not a PHP language feature or a library that the application must contact on every request. A deployed program normally needs the PHP runtime, its application files, the installed `vendor/` tree, and the generated autoloader. The Composer executable is not part of the request path; the generated autoloader runs within PHP.

## Mental Model

Think of Composer as the build-time coordinator between project intent and an installable PHP tree:

```text
project metadata + package sources
              ↓
        Composer commands
   select, fetch, install, inspect
              ↓
     vendor/ + generated files
              ↓
  application includes autoloader
              ↓
       PHP runs application
```

The project records which packages it needs. Repositories describe where package metadata and source or distribution archives can be found. Composer commands use that information to install package files and generate support files, including an autoloader. PHP then executes the application in its usual runtime; Composer is not present in the request path unless the application deliberately invokes it.

These are different concerns:

- **Package metadata** describes what the project requests and which packages are available. Chapters [93](093-dependency-resolution.md), [94](094-composer-json.md), [95](095-composer-lock.md), and [96](096-semantic-versioning.md) treat dependency selection, the project manifest, the lock file, and version constraints.
- **Installation** places the selected package code and generated support files on disk.
- **Autoloading** connects a class name used by application code to a loader that can find its file. This chapter shows the handoff; Chapters [97](097-autoloading.md) and [98](098-psr-4.md) explain the mechanisms.
- **Runtime execution** belongs to PHP and the application. PHP-FPM or another process manager, the PHP version, installed extensions, and configuration still matter after packages are installed. See the runtime chapters in Volume V.

## Core Concept: Dependency Manager and Project Tool

Composer's main role is dependency management. A package is a named unit of code with versioned metadata and a source or downloadable distribution. A project can request packages directly; those packages may request additional packages. The complete installed tree therefore contains both direct dependencies and the dependencies they rely on.

Packages have names such as `monolog/monolog`: a vendor or organization component and a package component. A package may publish multiple versions. Version selection rules and the algorithm that finds a compatible set are important enough to have their own chapters; here, treat them as inputs Composer processes.

Composer also provides project commands that sit around dependency management:

- validate project metadata before it is committed or released;
- install dependencies for local development, CI, or deployment;
- check whether the actual PHP runtime and extensions meet installed package requirements;
- report known dependency policy findings;
- regenerate the autoloader after relevant project changes;
- expose installed command-line tools under `vendor/bin/`.

This mix of responsibilities is why Composer appears in more than one phase of a project's lifecycle. It helps bootstrap a project, prepare a contributor's checkout, run local tools, build a deployment artifact, and review installed dependencies. Each command should have a clear place in that lifecycle.

## The Package and Repository Model

A repository is a source of package metadata. Composer's default public repository is Packagist. A project can also refer to private package repositories or version-control repositories, for example when an organization publishes internal libraries or temporarily uses a maintained fork. A repository describes packages and available versions; package metadata then identifies the code that Composer can download as a distribution archive or retrieve from source.

Repository configuration belongs to the root project. Composer does not recursively load repository declarations from each dependency. This keeps a library from silently adding arbitrary package sources to every application that uses it. In a company, centrally managed private repositories or a mirror can make package availability and credentials more consistent.

Repository ordering and identity have security consequences. If a package name can be satisfied by more than one source, repository priority can affect which source supplies it. Check the official [repository documentation](https://getcomposer.org/doc/05-repositories.md) when adding a source, especially for private packages or package-name overrides. Do not treat a package name alone as proof that code came from the intended maintainer.

Composer can install an application dependency or a developer tool. Project tools are generally installed with the project so the whole team can run the same local executable, for example:

```sh
./vendor/bin/phpunit
```

The package provides the tool; Composer creates a proxy in the project's `vendor/bin/` directory. This avoids requiring every contributor or CI runner to install a separately managed global copy. Global Composer packages are available for personal utilities, but they are not a substitute for declaring tools that a project needs to build and test consistently.

## A Core Command Workflow

Start in the project root, where Composer expects to find the project's metadata. A new project can begin with the interactive initializer:

```sh
composer init
```

For an existing project, adding a package is commonly done with `require`:

```sh
composer require psr/log
```

This records the requested dependency and then performs the dependency installation or update associated with the change. The `require` command is convenient because the metadata change and its installation are one operation. Review the resulting project files and run the test suite before committing.

When another contributor checks out the project, they normally install the dependencies already selected for that project:

```sh
composer install
```

For an application with a committed lock file, `install` reproduces the selected package versions represented there and builds the local `vendor/` tree. The difference between `install` and `update`, and the role of the lock file, are covered in [Chapter 95](095-composer-lock.md). The important workflow rule is that application development and CI generally use `install`; a deliberate dependency refresh is a separate change that should be reviewed and tested.

A common project check sequence is:

```sh
composer validate --strict
composer install --no-interaction
composer check-platform-reqs
composer audit
./vendor/bin/phpunit
```

The final command assumes the project has installed PHPUnit as a development tool; use the project's own test command if it differs. `validate --strict` treats metadata warnings as failures. `check-platform-reqs` compares installed package requirements to the actual PHP version and extensions available in the current environment. `audit` checks installed packages against configured dependency policies, which can include published security advisories and package status. These commands answer different questions: valid metadata does not prove application behavior; passing tests do not prove the production host has the necessary extension; and an audit report does not establish that all dependency code is safe.

`require` and `remove` are useful for intentional dependency changes. `update` asks Composer to resolve allowed versions and update the lock information. Its resolution behavior belongs in [Chapter 93](093-dependency-resolution.md); version-range design belongs in [Chapter 96](096-semantic-versioning.md). Avoid using `update` as a casual synonym for installing the project, because it can change selected dependency versions.

## The `vendor/` Tree and Generated Autoloader

The conventional install directory is `vendor/`. It contains third-party package files and Composer-generated support files. A simplified tree might look like this:

```text
project/
├── composer.json
├── composer.lock
├── public/
│   └── index.php
└── vendor/
    ├── autoload.php
    ├── bin/
    │   └── phpunit
    ├── composer/
    │   └── ... generated metadata and loader support ...
    ├── monolog/
    │   └── monolog/
    └── psr/
        └── log/
```

The exact directory content depends on the project and Composer configuration. The application entry point includes the generated loader before it uses classes supplied by installed packages:

```php
<?php

declare(strict_types=1);

require dirname(__DIR__) . '/vendor/autoload.php';

// Application bootstrap continues here.
```

Including `vendor/autoload.php` registers Composer's loader with PHP. When later code refers to an autoloadable class, PHP can ask registered autoloaders to load its definition. Composer generates the support code from project and package metadata; PHP performs the runtime class loading. Autoloading does not download a package, and including the file does not make an absent package appear. If `vendor/` was not built or is incomplete, the application can fail at runtime with a missing class or file.

The full generated tree is normally not committed to an application's version-control repository. The manifest and lock file describe the inputs; `composer install` builds `vendor/`. A deployment pipeline may construct `vendor/` during image or release creation and then ship that complete artifact. It must not copy application files while omitting `vendor/` if runtime code depends on it. Libraries intended for publication have different packaging considerations; follow the package's distribution model.

If project classes or autoload metadata change, `composer dump-autoload` regenerates autoloader files without performing a dependency update or full install. For a release build, Composer supports optimized autoloader generation:

```sh
composer install --no-dev --no-interaction --optimize-autoloader
```

`--no-dev` skips development packages and development-only autoload rules. `--optimize-autoloader` can reduce lookup work for deployed code, while requiring the generated result to match the code in that release. The official [optimization guide](https://getcomposer.org/doc/articles/autoloader-optimization.md) recommends optimization especially for production and explains its trade-offs. PSR-4 mapping and class resolution details remain in Chapters 97 and 98.

## Project Lifecycle and Environment Boundaries

Composer does its substantial work before the application serves requests: it reads metadata, contacts repositories when needed, downloads package archives or source, writes files, runs configured scripts and plugins, and generates autoload support. Installation cost depends on package metadata, the number and size of archives, network latency, decompression, filesystem speed, and project scripts. There is no useful single time-complexity figure for a real Composer build. Dependency resolution itself is treated in [Chapter 93](093-dependency-resolution.md).

At runtime, the application should use the already-built tree. A request should not run `composer install` to repair its dependencies. Doing so adds network and filesystem work to a user-facing path, creates races between concurrent requests, and may mutate code in a running release. Build the tree as a deployment artifact, validate it, and promote the complete artifact through environments.

The build environment and runtime environment must agree on platform requirements. A developer may have a different PHP minor version or extensions from the production container. Composer can be configured to resolve as if it were targeting a particular PHP platform, but that simulated setting does not install the real extension or prove that the deployed runtime matches it. `composer check-platform-reqs` deliberately checks the actual environment and ignores a simulated platform setting. Run it against the same image or platform that will execute the application.

Composer's `vendor/` directory also has a disk and deployment cost: it occupies the space of all installed package files plus generated metadata. PHP's runtime cost is separate. The autoloader adds class lookup and file inclusion work when classes are first used in a process; an optimized generated map can reduce some lookup work. OPcache and process lifetime affect how that work behaves in web and worker processes, as discussed in [Chapter 59](../04-php-under-the-hood/059-opcache.md) and [Chapter 71](../05-php-runtime/071-configuration.md). Measure application startup and deployment size before adopting more aggressive autoloader modes.

## Reliability and Security

Dependencies are code that executes with the privileges of the application or build process. Composer plugins can extend Composer's behavior, and project scripts can run commands during lifecycle events. The [Composer security guidance](https://getcomposer.org/doc/faqs/how-to-install-untrusted-packages-safely.md) explains that package installation and update can execute third-party code with the current user's access. Treat changes to dependencies, repositories, scripts, and plugins as executable-code changes in review. Use trusted sources, keep credentials scoped, avoid running Composer as a privileged account, and isolate unknown packages in a disposable environment.

Composer's `audit` command is useful for identifying packages matched by the configured policy sources, including known vulnerability advisories and other package policy findings. It is a point-in-time signal: a clean report cannot prove the package has no undisclosed vulnerability, malicious behavior, or unsafe configuration. Combine it with code and provenance review, prompt updates, tests, and a plan for responding when a finding affects production.

Do not suppress platform checks merely to make installation succeed on a mismatched machine. Flags that ignore requirements can produce an installed tree that cannot execute on the target PHP runtime. If a build intentionally targets another platform, make that target explicit and validate the actual runtime separately.

## Testing and Operational Checks

Composer commands are build operations, so verify both their declared inputs and the resulting application:

1. Run `composer validate --strict` when changing project metadata and in CI.
2. Build from a clean checkout with the same `install` command used by CI or deployment.
3. Run the test suite and static checks using the project's locally installed tools.
4. Run `composer check-platform-reqs` in the actual target image, including the right dev-package option for that stage.
5. Run `composer audit` as part of routine dependency review and define who evaluates and resolves findings.
6. For release builds, verify that `vendor/autoload.php` exists and that a minimal application bootstrap can load the classes it requires.
7. Promote the built artifact rather than mutating dependencies on a live host, and retain enough build metadata to reproduce or investigate it.

Network access and repository availability can fail during a build. A deployment design should build before the release window, cache or mirror package artifacts where appropriate, and fail clearly if required inputs cannot be retrieved. Do not silently continue with an old `vendor/` directory after a failed install: it may no longer correspond to the code or metadata being deployed.

## Common Mistakes

- **Installing packages manually.** This bypasses the project dependency workflow and makes future installs difficult to reproduce.
- **Running dependency updates during every deployment.** A deploy should build the reviewed dependency set. Updates are a separate change with code review and tests.
- **Committing only the application code and forgetting the installed tree at runtime.** Ensure the release artifact includes dependencies or builds them before serving traffic.
- **Assuming `vendor/autoload.php` downloads dependencies.** It only loads generated autoload support; installation must have already produced the files.
- **Using globally installed test tools in CI.** A global version can differ from the project's required version; prefer project-local tools.
- **Treating a passing `audit` command as a security certification.** The command reports known and configured findings, not every risk in third-party code.
- **Allowing arbitrary repositories and plugins without review.** They affect what code is installed and what can run during Composer commands.
- **Checking PHP requirements on a workstation only.** Verify them against the target runtime or container.
- **Copying an optimized autoloader between different code revisions.** Generated files must match the code tree being released.

## Senior Engineer Thinking

When adding a dependency, ask what problem it solves, whether its behavior belongs in the application, what it brings transitively, and which team owns upgrades. Check the package's maintenance, release process, license, platform needs, repository provenance, and runtime footprint. A small library can still expand the executable-code and maintenance surface.

When designing CI and release steps, separate dependency selection from dependency installation, test the same environment that will run the application, and make a release artifact immutable after validation. Keep private repository credentials out of source control. Decide how audits become actionable work instead of a noisy dashboard that nobody owns.

Finally, remember which layer owns a failure. Composer can report that an extension is missing, but cannot configure PHP-FPM to load it. An autoloader can locate a class file, but cannot repair application initialization order or an incompatible API. The PHP runtime executes the code; infrastructure configures that runtime; and application tests establish the behavior expected from both.

## Exercises

1. In a small PHP project, install one runtime library and one test tool locally. Record which Composer command created each entry and where the executable appears.
2. In a clean checkout, run the project's install command. Inspect the resulting `vendor/` tree and identify the generated autoloader and package files.
3. Add a minimal application entry point that requires `vendor/autoload.php`. Explain what fails if the dependency directory is deleted after deployment.
4. Create or use a CI container with an extension requirement different from your workstation. Compare the result of installation with `composer check-platform-reqs` on each environment.
5. Inspect a project's Composer scripts, plugins, and repositories. Document which operations may execute code and what credentials those operations can access.
6. Design a release pipeline that validates, installs, tests, audits, checks platform requirements, and packages an immutable artifact. Explain how it behaves if a repository is unavailable.

## Review Questions

1. Which work belongs to Composer at build time, and which work belongs to PHP at application runtime?
2. What role does a repository play, and why should repository configuration be reviewed?
3. What does `vendor/autoload.php` do? What does it not do?
4. Why is a project-local tool often preferable to a globally installed tool in CI?
5. How do `validate`, `check-platform-reqs`, and `audit` differ?
6. Why should a deployment install a previously reviewed dependency set instead of resolving new versions during rollout?
7. Which risks remain after a dependency audit reports no findings?
8. Why must Composer commands that execute scripts or plugins be run with appropriate user privileges and isolation?

## Summary

Composer manages PHP packages and the project workflow around them. It reads project metadata and repository information, installs the selected packages in the conventional `vendor/` tree, generates `vendor/autoload.php`, and exposes project tools under `vendor/bin/`. The application includes the generated loader and then runs under the normal PHP runtime.

Use project-local tools, validate metadata, install the reviewed dependency set, check the actual PHP and extension environment, and audit dependency policy findings. Treat repositories, plugins, scripts, and packages as part of the executable supply chain. Keep selection rules, manifest fields, lock-file guarantees, version constraints, and autoloading mechanisms distinct; later chapters develop each in turn.

## References

- [Composer command-line interface](https://getcomposer.org/doc/03-cli.md)
- [Composer basic usage](https://getcomposer.org/doc/01-basic-usage.md)
- [Composer repositories](https://getcomposer.org/doc/05-repositories.md)
- [Composer autoloader optimization](https://getcomposer.org/doc/articles/autoloader-optimization.md)
- [Composer scripts](https://getcomposer.org/doc/articles/scripts.md)
- [Composer plugins](https://getcomposer.org/doc/articles/plugins.md)
- [Installing untrusted packages safely](https://getcomposer.org/doc/faqs/how-to-install-untrusted-packages-safely.md)
- [Chapter 93 — Dependency Resolution](093-dependency-resolution.md)
- [Chapter 94 — `composer.json`](094-composer-json.md)
- [Chapter 95 — `composer.lock`](095-composer-lock.md)
- [Chapter 96 — Semantic Versioning](096-semantic-versioning.md)
- [Chapter 97 — Autoloading](097-autoloading.md)
- [Chapter 98 — PSR-4](098-psr-4.md)
