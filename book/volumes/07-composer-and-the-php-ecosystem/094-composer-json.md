---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 94
title: composer.json
slug: composer-json
status: complete
summary: ../../_ai/chapter-summaries/094-composer-json-summary.md
---

# Chapter 94 — composer.json

## Why This File Matters

`composer.json` is the project’s declaration of intent. It says what the project is, which PHP packages and platform capabilities it needs, how its own classes are loaded, and which repeatable commands its maintainers use. Composer reads this manifest to install and manage a project. It is also a contract with other developers and, for a published library, with downstream applications.

That contract has limits. The manifest describes acceptable dependency versions; it does not by itself record the exact versions selected for one installation. Composer records that selected set in `composer.lock`, covered in [Chapter 95](095-composer-lock.md). The distinction matters when reviewing a dependency change: one file states policy, the other records a concrete resolution. [Chapter 93](093-dependency-resolution.md) explains how Composer finds a set that satisfies the policy.

This chapter treats the file as structured configuration with operational consequences. A syntactically valid JSON file can still describe the wrong PHP requirement, omit a needed extension, map a namespace to a missing directory, or run a command that fails in CI. Validation is necessary, but it cannot replace tests and review.

## The Root Manifest

Composer operates on a root package: the project where a Composer command is run. The root `composer.json` controls that project’s dependencies and configuration. Some fields are meaningful only at the root, including `require-dev`, `autoload-dev`, `scripts`, and `config`. A dependency’s manifest cannot inject its scripts or development dependencies into the consuming application. This lets an application define its own build and test workflow rather than inheriting those commands from every library it uses.

The manifest is JSON, so it has no comments and does not allow trailing commas. Use a trailing-comma-free example and validate it with Composer rather than relying on an editor’s generic JSON check alone. Composer validates against its schema and can also report issues that matter for publishing or consistency with an existing lock file. The full field reference is the [Composer schema documentation](https://getcomposer.org/doc/04-schema.md); the machine-readable [Composer schema](https://getcomposer.org/schema.json) is useful to editors and tooling.

## Describe the Package Honestly

The optional package `name` is written as a lowercase vendor and package name, such as `acme/billing`. Published libraries need a package name; it identifies the package in repositories and dependency declarations. A human-readable `description` explains its purpose. `license` communicates the package's distribution terms. Choose an identifier that accurately describes this package; review dependency licenses separately.

The `type` field tells Composer what kind of package it is. The default is `library`, which is appropriate for reusable code. An application can set `"type": "project"` to describe a deployable project. Custom package types matter when an installer or plugin has behavior associated with that type; inventing a type name alone does not add installation behavior.

Most projects should omit the `version` field. Composer generally determines versions from version-control tags and branches. Hard-coding a version in the manifest creates another value that can disagree with repository history. The [schema documentation](https://getcomposer.org/doc/04-schema.md#name) describes the package identity and metadata fields, including when they are required for publication.

## State Runtime and Package Requirements

The `require` object declares packages and platform packages needed for the project to work. Platform packages include PHP itself and extensions. For example, `"php": "^8.3"` expresses a PHP compatibility range, while `"ext-pdo": "*"` requires the PDO extension to be present. An extension requirement is easy to overlook because a developer’s local machine may have more extensions than a production container.

Declare requirements that the application truly uses. If code calls an extension API directly, declare the extension instead of assuming it will be installed as a side effect of another package. If there are separate environments, compare their PHP and extension capabilities in deployment checks. Composer’s platform packages and version constraints are documented in [Package links](https://getcomposer.org/doc/04-schema.md#package-links) and [platform packages](https://getcomposer.org/doc/04-schema.md#platform-packages).

`require-dev` names packages needed by maintainers, such as test runners and static analyzers. Composer uses those packages in a development install; production installs can omit them with `--no-dev`. Keep application runtime imports out of `require-dev`: the production application must not rely on a package that disappears when dev dependencies are excluded.

Constraints in `require` and `require-dev` express compatibility intent; they are not an exact installation record. When Composer updates the dependency set, the solver chooses versions allowed by the constraints, and the lock file records the result. See [Chapter 93](093-dependency-resolution.md) for the solving process and [Chapter 95](095-composer-lock.md) for reproducible installs.

## Teach Composer How to Load Your Code

The `autoload` section maps the project’s classes to source files. The following common PSR-4 mapping associates the `Acme\\Billing\\` namespace prefix with the `src/` directory:

```json
{
  "autoload": {
    "psr-4": {
      "Acme\\Billing\\": "src/"
    }
  }
}
```

Because JSON uses backslashes as escape characters, each PHP namespace separator is written as `\\` in the JSON string. After installing dependencies, application code normally loads Composer’s generated `vendor/autoload.php` once at its entry point. When an autoload mapping changes, regenerate the autoloader with `composer dump-autoload` (or through the normal install/update operation). A successful schema check does not prove the directory and classes actually match the mapping.

`autoload-dev` can map test namespaces and other development-only classes. This keeps test code out of the autoload rules exported to consumers of a library. The available mechanisms include PSR-4, PSR-0, class maps, and files; choose based on the code’s loading needs rather than adding mappings speculatively. See [Chapter 97](097-autoloading.md) for Composer autoloading and [Chapter 98](098-psr-4.md) for the detailed PSR-4 rules. This chapter uses only the manifest-level view.

## Make Routine Work Repeatable

The root `scripts` object gives names to project commands and hooks. A script can be a command string or an array of commands. When the value is an array, commands run in order. A project can also refer to another named script with the `@name` form. Composer executes the root package’s scripts; scripts declared by dependencies are not run as part of the root project’s script lifecycle. See the official [Composer scripts documentation](https://getcomposer.org/doc/articles/scripts.md) for supported events and callback forms.

For example, a `test` script makes the expected test command easy to find and invoke consistently:

```json
{
  "scripts": {
    "test": "phpunit",
    "analyse": "phpstan analyse",
    "check": ["@test", "@analyse"]
  }
}
```

This assumes the corresponding development tools are installed and available in the project. The script name does not install a tool or make a failing command pass; it provides a shared entry point for contributors and CI. Treat project scripts as executable code: inspect changes to them, make their behavior clear, and avoid placing credentials in command strings. Plugins are a separate mechanism from scripts, though both can affect Composer operations.

## Configure Composer Deliberately

The root `config` object adjusts Composer behavior. A few settings are especially useful to understand:

- `sort-packages` asks Composer to keep dependency entries sorted when it edits the manifest. This is a maintenance aid, not a change to runtime behavior.
- `platform` can make dependency resolution behave as though Composer were running on a specified PHP or extension platform. It helps resolve against a deployment target when development machines differ, but it does not change the PHP binary that executes the application.
- `allow-plugins` controls which Composer plugins may run. Since Composer 2.2, plugins require explicit permission by default. Allow only plugins the project intentionally uses and trusts; permitting all plugins removes that safeguard.

The [`config` reference](https://getcomposer.org/doc/06-config.md) describes the available keys and their defaults. A simulated platform is not proof that production satisfies the platform requirements. After dependencies are installed in the actual deployment environment, `composer check-platform-reqs` checks the real PHP and extension requirements and ignores the simulated `config.platform` values. This catches cases where the solver was given a target different from the runtime.

## A Small Application Manifest

Here is a coherent example for a private application. The dependency constraints are illustrative; choose constraints that match the project’s support policy and available package releases.

```json
{
  "name": "acme/billing-app",
  "description": "A small billing application.",
  "type": "project",
  "license": "proprietary",
  "require": {
    "php": "^8.3",
    "ext-pdo": "*",
    "psr/log": "^3.0"
  },
  "require-dev": {
    "phpunit/phpunit": "^11.0",
    "phpstan/phpstan": "^2.1"
  },
  "autoload": {
    "psr-4": {
      "Acme\\Billing\\": "src/"
    }
  },
  "autoload-dev": {
    "psr-4": {
      "Acme\\Billing\\Tests\\": "tests/"
    }
  },
  "scripts": {
    "test": "phpunit",
    "analyse": "phpstan analyse",
    "check": ["@test", "@analyse"]
  },
  "config": {
    "platform": {
      "php": "8.3.0"
    },
    "sort-packages": true,
    "allow-plugins": {}
  }
}
```

An empty `allow-plugins` map means no Composer plugins are permitted. Add an entry only when the project needs a particular plugin and the team has reviewed it. A private application should also avoid publishing private repository credentials in the manifest; use Composer’s [authentication mechanisms](https://getcomposer.org/doc/articles/authentication-for-private-packages.md) and the deployment environment’s secret management instead.

## Validate the Manifest and the Environment

Validation should happen at several layers because each command answers a different question:

```sh
# Check JSON/schema and publish-related warnings before a lock file exists.
composer validate --strict --no-check-lock

# With an existing lock file, also check that it matches the manifest.
composer validate --strict

# Rebuild Composer's generated autoloader after changing mappings.
composer dump-autoload

# Run the project's repeatable checks.
composer check

# In the deployment PHP environment, verify actual platform requirements.
composer check-platform-reqs
```

`--no-check-lock` is appropriate when there is no lock file to compare yet. Once the project has a lock file, ordinary validation is useful because it catches a manifest/lock mismatch. Do not routinely suppress publishing checks for a package that is about to be released. `composer validate` can detect malformed JSON, schema problems, and certain consistency or publication issues, but it cannot prove the source code passes tests, the declared extensions are used correctly, or the deployment image has been configured well.

For production deployment, install from the reviewed lock file with development dependencies omitted when appropriate, and consider Composer’s optimized autoloader option. Follow that with application tests or smoke checks in the target environment. For libraries, run the test and compatibility checks against the PHP versions the library claims to support. The manifest tells tools and collaborators what the project expects; CI and deployment checks establish whether those expectations hold.

## Common Maintenance Failures

An overly loose or overly narrow PHP requirement can both cause trouble. A requirement that is too narrow rejects valid environments; one that is too broad can let Composer select packages incompatible with the runtime. State support based on the PHP versions actually tested and deployed. The same principle applies to extension requirements.

Putting a runtime package in `require-dev` can leave production without a class or function the application needs. Conversely, putting test-only tooling in `require` makes production installs larger and exposes tools unnecessarily. Reviewing the intended environments alongside each dependency helps keep the boundary accurate.

A namespace map can be valid JSON but point to the wrong directory; test a real class load. A script can exist but refer to an unavailable executable; run it in CI. A `config.platform` declaration can mask a production mismatch; check the deployed runtime with `composer check-platform-reqs`. An unreviewed plugin can run code during Composer operations; keep plugin permissions narrow. Finally, hand-editing only the manifest can leave the lock file out of sync. Use a deliberate dependency update workflow and review the lock-file change as described in [Chapter 95](095-composer-lock.md).

## Exercises

1. Add a PHP extension that the project directly uses. Run validation, then check whether the development and deployment environments both provide it.
2. Add a test namespace under `autoload-dev`. Write a small test that loads a test class through Composer’s autoloader, then remove the mapping and observe the failure.
3. Add a `check` script that runs a formatter check and static analysis. Run the script locally and in CI, and make sure both environments use the same command.
4. Temporarily set `config.platform.php` to a version newer than the PHP binary used to run the application. Compare dependency validation with `composer check-platform-reqs` and explain what each result tells you.

## Review Questions

1. What is the difference between the constraints in `composer.json` and the exact dependency versions recorded in `composer.lock`?
2. Why should runtime dependencies belong in `require` rather than `require-dev`?
3. What problem does `config.platform` solve, and why must it be followed by checking the real deployment platform?
4. Why are Composer plugins subject to an explicit allow-list, and how are they different from root scripts?
5. What can `composer validate` establish, and what requires tests or an environment check?

## Summary

`composer.json` declares package identity, runtime and development requirements, autoload rules, scripts, and Composer configuration for the root project. Keep each field aligned with how the code is built and deployed. Use Composer’s schema validation, project tests, and actual platform checks together. Treat the manifest as compatibility policy and operational configuration; use the lock file for the concrete dependency set and consult the later autoloading chapters for detailed mapping behavior.

## References

- [Composer schema](https://getcomposer.org/doc/04-schema.md)
- [Composer configuration](https://getcomposer.org/doc/06-config.md)
- [Composer scripts](https://getcomposer.org/doc/articles/scripts.md)
- [Composer CLI](https://getcomposer.org/doc/03-cli.md)
