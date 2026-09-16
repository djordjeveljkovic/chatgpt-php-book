---
book: The Complete Modern PHP Engineering Book
volume: 10
volume_title: SECURITY
chapter: 156
title: Dependency Security
slug: dependency-security
status: complete
summary: ../../_ai/chapter-summaries/156-dependency-security-summary.md
---

# Chapter 156 — Dependency Security

A dependency is code that enters your process with your privileges. A Composer package can read requests, access the database, send network traffic, execute commands, and run during installation. Dependency security is therefore part of application security, not only a package-management concern.

The goal is not to avoid all third-party code. It is to know what is installed, why it is needed, which versions are allowed, how vulnerabilities are assessed, and how a safe update reaches production.

## Why this matters

A vulnerable package may expose an endpoint, parse attacker-controlled input, or provide a gadget chain for unsafe deserialization. A malicious or compromised package can behave correctly in tests and act during installation or only under a rare input. Transitive dependencies are part of the attack surface even when no application code imports them directly.

Composer resolves a dependency graph. `composer.json` describes acceptable constraints; `composer.lock` records the selected graph. Commit the lock file for applications and deploy the locked graph consistently. A lock file improves repeatability, but it does not prove that every selected package is safe.

## Keep the graph visible

Review direct and transitive packages, their licenses, abandoned status, maintenance activity, and access to sensitive data. Use Composer's built-in validation and audit commands in CI:

```text
composer validate --strict
composer audit
composer install --no-dev --prefer-dist --no-interaction
```

Use the exact command and options supported by the Composer version in the build image. Run the audit against the graph that will ship; a separate developer graph can contain dev-only packages that are not present in production, while a production graph can have a different platform PHP version.

Do not solve every advisory by blindly upgrading all packages. First identify the affected package and code path, the vulnerable feature, the reachable input, and the available fixed version. A package may be exploitable only when a particular parser or integration is enabled, but that conclusion needs evidence and a compensating control.

## Constraints and updates

Use version constraints that communicate policy, then review the resulting lock diff. A routine update can change many transitive packages and generated autoload files. Inspect the diff, run tests and static analysis, and deploy the artifact produced by that build.

```json
{
    "require": {
        "php": ">=8.2",
        "vendor/http-client": "^7.4"
    },
    "config": {
        "allow-plugins": {
            "vendor/trusted-plugin": true
        }
    }
}
```

The example allows one named Composer plugin. Composer plugins execute code in the Composer process and can affect installation, so do not set a broad wildcard merely to silence a prompt. The package name and allowed plugin policy must be reviewed together; a transitive plugin may require an explicit decision.

A constraint such as `^7.4` is not a promise that every release is safe. Review release notes, advisory data, API changes, and lockfile changes. For reproducible production builds, use `composer install` with the committed lock file rather than running an unconstrained update on the server.

## Advisories and response

Track vulnerability advisories from Composer's audit data, the package maintainer, and relevant ecosystem sources. Classify a finding by affected version, reachability, exposure, exploitability, and business impact. Then choose an action with an owner and deadline:

- update to a fixed version and test the affected path;
- disable or remove the feature that reaches the vulnerable code;
- add a bounded compensating control while an update is prepared;
- replace or fork an abandoned package when the risk is accepted only temporarily;
- isolate and rotate credentials if the vulnerable code may have read them.

A false positive still deserves a recorded reason and a review date. An audit command is a detector, not a risk decision. Conversely, a clean audit does not detect every malicious release, logic flaw, or vulnerability without a published advisory.

For an emergency, build a new artifact from a reviewed lock change, test the affected behavior, deploy with a rollback plan, and monitor errors and security signals. Do not edit `vendor/` directly on a production host; that creates an untracked state that the next deploy will overwrite.

## Package code and runtime boundaries

Review packages that parse files, process HTML/XML, invoke processes, handle cryptography, access cloud metadata, or register framework middleware. Keep sensitive operations behind application interfaces so replacing a package or adding policy does not require changing every handler. Do not assume a package is harmless because it has no network code; any PHP code in the process shares the process's privileges.

Separate development tools from production dependencies with `require-dev` and deploy with `--no-dev` when appropriate. This reduces attack surface and image size, but it does not protect a developer workstation or CI runner where dev dependencies execute. Run static analysis and tests in a controlled build environment.

## Testing and verification

Test the lockfile and package policy in CI:

1. Fail when `composer.lock` is missing or inconsistent with `composer.json`.
2. Run the audit command against the locked graph.
3. Build with a fixed PHP and Composer toolchain, then run tests, static analysis, and license or policy checks.
4. Record the package graph and build identifier with the release.
5. Exercise high-risk package boundaries with integration tests.
6. Verify that production contains only the intended vendor tree and does not execute install-time development hooks unexpectedly.

Use a review checklist for new dependencies: purpose, maintainer and release history, transitive graph, license, advisory history, permissions, update cadence, and removal plan. A small package with a narrow purpose is easier to review than a large package imported for one helper, but size alone is not a security verdict.

## Exercises

1. Add a CI job that validates the lock file and runs `composer audit` against the production graph.
2. Choose a transitive package with an advisory. Trace which application path reaches it and write a temporary compensating control.
3. Review a Composer plugin policy and explain why each allowed plugin is necessary.

## Review questions

- Why is `composer.lock` necessary but insufficient for dependency security?
- What risks are introduced by Composer plugins?
- How should a team distinguish an advisory from an exploitable application path?
- Why should production install from a reviewed artifact rather than run `composer update`?
- What remains outside the coverage of an advisory database?

## References

- [Composer documentation: audit](https://getcomposer.org/doc/03-cli.md#audit)
- [Composer documentation: basic usage and lock files](https://getcomposer.org/doc/01-basic-usage.md)
- [Composer documentation: allow-plugins](https://getcomposer.org/doc/06-config.md#allow-plugins)
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [OWASP Software Component Verification Standard](https://scvs.owasp.org/)
