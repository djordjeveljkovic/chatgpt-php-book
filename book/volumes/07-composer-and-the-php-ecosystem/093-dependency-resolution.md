---
book: The Complete Modern PHP Engineering Book
volume: 7
volume_title: COMPOSER AND THE PHP ECOSYSTEM
chapter: 93
title: Dependency Resolution
slug: dependency-resolution
status: complete
summary: ../../_ai/chapter-summaries/093-dependency-resolution-summary.md
---

# Chapter 93 — Dependency Resolution

## Why This Matters

A PHP application rarely installs only the packages named directly in its `composer.json`. A web framework can require a console component, which in turn requires a contracts package and several polyfills. Each dependency brings its own version constraints and platform needs. Composer must choose one compatible package version for the project, including all of those indirect requirements.

When that choice cannot be made, the error is a useful description of incompatible requirements, not a request to delete `composer.lock` or install the newest version of everything. When a choice can be made, the selected set is a build input: a small change in a version constraint, PHP platform, repository metadata, or update command can change it. Understanding the resolution boundary helps a team make that change deliberately and reproduce it in CI and production.

## Mental Model

Treat the project as the root of a package graph:

```text
application
├── logger ^3.0
│   └── logging-contract ^2 || ^3
└── http-client ^7.0
    └── logging-contract ^3
```

Every edge states a requirement on a package. Composer gathers candidate package versions from its configured repositories and selects a set that satisfies the root requirements, transitive requirements, conflicts, stability rules, and platform requirements together. The example graph can select one `logging-contract` version only if the two incoming ranges have a common member.

This graph is a reasoning aid, not a description of Composer's internal solver algorithm. Dependency relationships can contain cycles; Composer does not simply install packages by walking a topological order. The important question is whether there is a complete compatible assignment of package versions and platform facts. Composer's [version documentation](https://getcomposer.org/doc/articles/versions.md) describes how it matches constraints against available package versions.

## What Counts as a Candidate

A package name and a constraint describe the versions a package consumer will accept. A constraint is not itself an installed version. For example, `^3.2` allows a set of versions; a lock file later records the particular version/reference selected for the project. The exact caret, tilde, comparison, and wildcard semantics belong to [Chapter 96 — Semantic Versioning](096-semantic-versioning.md), so this chapter focuses on how constraints interact.

Composer can discover candidate versions from tags and development branches in VCS repositories or metadata from package repositories. The root project's `repositories` configuration determines custom repositories; Composer does not recursively load repository declarations from dependencies. The default repository is Packagist. See the official [Composer repositories guide](https://getcomposer.org/doc/05-repositories.md) for repository types and priority, and [Chapter 94 — composer.json](094-composer-json.md) for the broader manifest field reference.

The graph can include more than ordinary package names. A package may declare that it conflicts with another package range, offers a virtual capability with `provide`, or replaces another package with `replace`. Those relationships affect whether a selection is valid. A conflict does not mean “try this version later”; it excludes combinations that would install incompatible packages together. Providers and replacements can satisfy a requirement through an alternate package. Treat these declarations as package-maintainer promises: an incorrect `provide` or overly broad `replace` can make a graph appear installable while leaving the application without the behavior it expects. The [Composer schema reference](https://getcomposer.org/doc/04-schema.md#package-links) defines these links.

Composer does not guess which API your application uses or prove that two versions are behaviorally compatible. It can only enforce declared constraints and metadata. A package that incorrectly claims compatibility may install successfully and still fail at runtime or break tests.

## A Constraint Intersection

Imagine a project requiring `acme/renderer:^2.0` and `acme/reports:^4.0`. Both packages require `psr/log`, but the renderer requires `^2.0` and reports requires `^3.0`. In a graph view:

```text
root ──requires──> acme/renderer ^2.0 ──requires──> psr/log ^2.0
  └──requires──> acme/reports ^4.0 ──requires──> psr/log ^3.0
```

If the only available `psr/log` versions accepted by the first edge are outside those accepted by the second, Composer cannot choose a version satisfying both. The root has not requested `psr/log` directly, but the transitive requirements still constrain the solution. Possible remedies depend on the facts: one upstream package may have a newer compatible release; one of the root constraints may be unnecessarily narrow; the project may need an intentional upgrade of a related package set; or the dependencies may truly be incompatible and require replacing one.

Adding a direct `psr/log` requirement does not erase either transitive constraint. It adds another edge to the graph. Nor does forcing an exact version make incompatible ranges intersect. First identify which package introduced each constraint, then change a requirement only when the application's actual compatibility contract permits it.

## Stability and Selection Policy

Composer classifies versions as `dev`, `alpha`, `beta`, `RC`, or `stable`. For root projects, `minimum-stability` defaults to `stable`; versions below that threshold are filtered out unless a package-specific stability flag allows them. `prefer-stable` is a preference for stable releases when compatible alternatives exist. It does not override a required development constraint, and it cannot make an unstable version acceptable when the minimum-stability rules exclude it. These settings affect the candidate set and priority, not the meaning of your PHP code. See the official [schema documentation for minimum-stability and prefer-stable](https://getcomposer.org/doc/04-schema.md#minimum-stability).

Composer generally selects the highest matching compatible versions among the candidates under consideration. “Highest” does not mean “latest version on every package independently”: every selection must still satisfy the whole graph, root constraints, stability and platform rules, lock/update scope, repository metadata, and any configured dependency policies. If a package version does not appear to be selected, inspect its constraints and the update scope instead of assuming that the solver is malfunctioning.

Current Composer releases also apply dependency security policies during package operations. Composer 2.9 introduced default blocking of versions with active security advisories during updates. Composer 2.10 introduced unified `config.policy` settings and blocks malware-flagged versions during `update`, `require`, and `remove`, as well as during `install` by default. This may look like a version conflict even if the declared ranges overlap. Inspect `composer audit` and the root project's policy settings before changing requirements. The [Composer policy documentation](https://getcomposer.org/doc/06-config.md#policy) describes the current controls; the [Composer changelog](https://github.com/composer/composer/blob/main/CHANGELOG.md) records when version-specific behavior was introduced. In Composer 2.10 and later, `--no-blocking` disables all dependency-policy blocking for that command; Composer 2.9 used the older `--no-security-blocking` option. These flags change security policy, not package compatibility, so use them only as deliberate, reviewed exceptions.

## Platform Requirements

The PHP interpreter, extensions, selected system libraries, and Composer APIs can participate in the dependency graph as virtual platform packages. Examples include `php`, `ext-intl`, `lib-curl`, `composer-plugin-api`, and `composer-runtime-api`. A package that requires PHP `^8.2` cannot be selected for a platform modeled as PHP 8.1, even if the package itself is available. Composer's [platform dependencies guide](https://getcomposer.org/doc/articles/composer-platform-dependencies.md) documents the virtual package names and their behavior.

By default, Composer sees the PHP process that runs Composer and its installed extensions. A team can configure `config.platform` in the root project to model a deployment target while resolving from a different developer machine. This makes the modeled platform part of the resolution contract, but it does not change the developer's interpreter and does not install missing extensions. The configuration guide warns that a package can be selected for a fake PHP version and then fail on the actual PHP runtime. Run `composer check-platform-reqs` in the deployment environment: this command checks the real PHP and extension versions and ignores the emulated `config.platform` values.

`--ignore-platform-reqs` can help diagnose whether a platform constraint is the only obstacle to a resolution. It does not make incompatible PHP syntax, extension calls, or runtime behavior work. Do not make it part of a normal deployment command to silence a mismatch. Prefer resolving against the production PHP/extensions or declaring a deliberate target in `config.platform`, then checking the actual deployment platform.

Composer's platform view is also not a full operating-system or native-library compatibility proof. Packages may depend on behavior beyond declared `ext-*` or `lib-*` constraints. Test the built application in an environment representative of production.

## `install` and `update` Have Different Jobs

For an application with a committed lock file, use `install` to materialize the versions already selected for that project:

```sh
composer install
```

Composer reads the lock file and installs the recorded package set into `vendor/`. If there is no lock file, `install` has no selected set to reproduce: Composer resolves the current constraints and writes a new lock file. That is a fresh resolution and can select different versions at a later date.

Use `update` when you intend Composer to resolve a new set of versions and write it to the lock file:

```sh
# Resolve all packages allowed by the current manifest and platform.
composer update

# In a project that already requires this package, preview its update set.
composer update monolog/monolog -W --dry-run
```

An update also installs the resulting packages unless told otherwise. A targeted update limits the change set; `-W` allows dependencies of the named package to move, including dependencies that are also root requirements. A partial update can still be constrained by other locked packages. Review the proposed lock changes, remove `--dry-run` only when the intended update is understood, then run the project's tests. The [CLI reference for `install` and `update`](https://getcomposer.org/doc/03-cli.md#install-i) documents current command behavior and options.

If a manifest requirement changes while a lock file exists, `install` is not a substitute for resolving that change. Check whether the lock still matches the manifest with `composer validate`; use an intentional `composer update` for the affected package set when the lock needs a new selection. Avoid running an unrestricted update merely because one constraint changed: unrelated packages can move within their declared ranges and enlarge the review surface.

For applications, committing `composer.lock` makes the resolved graph available to teammates, CI, and deployment. A reusable library normally publishes its constraints in `composer.json`; its own lock file does not pin versions for applications consuming it. For the lock file's recorded data and merge behavior, see [Chapter 95 — composer.lock](095-composer-lock.md).

## Reproducibility Boundaries

A lock file is a record of the resolved package choices, not a complete container image. It helps ensure that repeated installs use the same selected package versions and references, but the result still depends on the actual platform, Composer behavior, repositories and artifact availability, installation plugins, scripts, and any other build steps. An update run can also observe newly published repository metadata. Composer's [basic-usage documentation](https://getcomposer.org/doc/01-basic-usage.md) describes why applications commit the lock file and install from it; deployment must still control its execution environment.

Use the following boundary:

```text
composer.json constraints + repository metadata + platform model
                         ↓ update
                    composer.lock
                         ↓ install
             vendor files + generated autoloader
```

The lock controls the package selection consumed by `install`. It does not freeze the PHP binary, extensions, OS packages, environment variables, plugin/script side effects, or application configuration. Make the PHP and extension target explicit, pin the Composer version used in CI/deployment when tool behavior is part of the build contract, and run platform checks and tests in an equivalent environment.

Plugins and scripts execute code during Composer operations. Only enable trusted plugins, review script definitions and package provenance, and treat dependency updates as code changes that need review. A deterministic version choice is not a security review, and a lock file does not establish that a package is safe.

## Reading Resolution Errors

Composer's error output usually presents a chain of constraints that cannot be satisfied together. Read it from the root requirement through the dependent package to the blocked package or platform. Look for:

- a version range that excludes the only compatible upstream release;
- a package conflict that rules out an otherwise matching candidate;
- a minimum-stability rule that removes a candidate;
- a current Composer dependency policy that blocks an advisory-affected or malware-flagged candidate;
- a PHP or extension requirement that does not match the actual or modeled platform;
- a package absent from configured repositories or not yet visible in their metadata;
- an update scope that leaves a related package fixed at its locked version.

Composer includes commands that answer different diagnostic questions:

```sh
# Inspect the installed dependency tree.
composer show --tree

# Who requires a package in this project?
composer depends psr/log -t

# What blocks a target package version or PHP version?
composer prohibits psr/log 3.0.0 -t
composer prohibits php 8.4

# Check the manifest/lock relationship and the host environment.
composer validate
composer check-platform-reqs
composer audit
composer diagnose
```

`depends` (also named `why`) explains which installed packages require a package. `prohibits` (also named `why-not`) asks which constraints block a target version, including a platform version. `validate` checks manifest and lock validity/freshness; `check-platform-reqs` checks the real installed platform; `diagnose` checks common Composer configuration and environment problems. None of these commands can decide whether your application is actually compatible with a broader constraint—that requires code review and tests. These diagnostics are documented in the [Composer CLI reference](https://getcomposer.org/doc/03-cli.md#depends-why) and [troubleshooting guide](https://getcomposer.org/doc/articles/troubleshooting.md).

For a transient conflict, try a dry run of a narrow update and inspect the proposed changes before writing a new lock file:

```sh
composer update vendor/package -W --dry-run -vvv
```

Verbose output can expose repository or constraint details, but it may also include internal URLs or environment-specific information; redact logs before sharing them. If the package is missing, verify its exact name, version/tag, configured root repository, and stability threshold before clearing caches or changing unrelated constraints.

## Failure, Recovery, and Testing

Treat a dependency update like a code change. A practical review path is:

1. Record the desired outcome: a feature, security fix, or platform upgrade.
2. Run `composer validate` and inspect the existing dependency tree.
3. Use `composer prohibits` or the solver error to find the narrowest incompatible requirement.
4. Make only a constraint/platform/update-scope change that reflects tested compatibility.
5. Preview a targeted update with `--dry-run`, then write the resulting lock file intentionally.
6. Review the lock diff for unexpected package movement and run unit, integration, static-analysis, and application-startup checks.
7. From a clean checkout, run `composer install` and `composer check-platform-reqs` in the deployment-like environment.

If the solver reports that no solution exists, preserve the committed lock file and fix the incompatible requirement instead of deleting the lock. A different failure can happen after Composer has found a solution: downloads, installation, or scripts may fail after the lock file or part of `vendor/` has changed. Inspect `git diff -- composer.lock`, decide whether to keep or restore the proposed lock, then run `composer install` to reconcile `vendor/` with that lock. If a lock-file merge is conflicted, resolve the manifest constraints and let Composer generate a compatible lock through an update, then validate and test it. [Chapter 95](095-composer-lock.md) covers lock-file review in more depth.

Useful tests include a clean install in CI, a platform check on every supported runtime image, and tests of package integrations that exercise the APIs your application calls. For a library, test against the range of dependency versions you claim to support rather than trusting your own development lock file alone. Dependency resolution can establish that metadata is mutually consistent; only execution tests can reveal undeclared incompatibility.

## Common Mistakes

- Assuming each direct dependency is resolved independently from its transitive dependencies.
- Treating a version constraint as the version Composer will install.
- Assuming the newest release of every package is necessarily compatible with the complete graph.
- Adding a direct requirement to “force” a version without checking the constraints that already apply.
- Lowering `minimum-stability` globally to make one development package appear.
- Treating a security-policy block as an ordinary version-range conflict or disabling it without review.
- Using `--ignore-platform-reqs` as a permanent deployment fix.
- Running a full `composer update` to repair one package and accepting unrelated lock changes without review.
- Deleting `composer.lock` from an application and resolving against whatever repository metadata is current.
- Assuming `config.platform` changes the runtime PHP or verifies production extensions.
- Treating a successful install as proof that packages are behaviorally or security compatible.
- Believing the lock file controls OS libraries, scripts, plugins, or PHP configuration.

## Senior Engineer Thinking

When resolution fails, ask which statement is false or too restrictive: the application's direct requirement, a library's published compatibility range, the modeled PHP/extensions, the repository's available versions, or the update scope. Do not widen a constraint unless tests and supported environments justify the wider contract. For a library, broad constraints help consumers combine packages; for an application, a lock file records the tested selection.

When resolution succeeds, ask what the result says and what it cannot say. It says Composer found a set consistent with the metadata it considered. It does not say that every runtime environment has the required extension, that the package code works with your call sites, or that the package should be trusted. Build reproducibility comes from combining a reviewed lock file with controlled tooling, platform checks, trusted install hooks, and tests.

## Exercises

1. Draw the package graph for a small application with two direct dependencies that both require the same transitive package. Give each edge a version range and show one compatible intersection and one unsatisfiable pair.
2. In a disposable project, require `monolog/monolog:^3.0`, generate a lock file, and inspect the tree. Identify one transitive package and trace which requirement introduced it.
3. Configure a project to emulate an older PHP target with `config.platform`. Explain which checks use the emulated version and which command checks the real interpreter.
4. Start with a compatible installed project and use `composer prohibits` to find why a proposed package or PHP version cannot be installed. Record the complete chain, then propose the smallest valid change.
5. Compare `composer install`, a full `composer update`, and a targeted update with `-W`. State which artifacts each command can change and what should be reviewed.
6. Design a CI test for a library that declares support for multiple versions of a dependency. Explain why one committed development lock file cannot prove the entire supported range.
7. List three sources of build variation that a lock file does not control, and choose a test or operational check for each.

## Review Questions

1. How do transitive requirements constrain a package even when the root project does not require it directly?
2. Why is dependency resolution a graph-wide compatibility problem rather than a series of independent “latest version” lookups?
3. What do `minimum-stability` and `prefer-stable` each control?
4. What are platform packages, and why does `config.platform` not prove the production runtime satisfies them?
5. When should a project use `install`, and when should it use `update`?
6. What can a lock file reproduce, and which build inputs remain outside its boundary?
7. How do `depends` and `prohibits` differ when diagnosing a constraint conflict?
8. Why does successful resolution not prove behavioral compatibility or package safety?

## Summary

Composer resolves the root project's direct and transitive package requirements as one compatibility problem. Candidate versions must satisfy all applicable constraints, conflicts, stability rules, repository availability, and the modeled PHP/extensions/platform requirements. A successful selection enforces declared metadata; tests are still needed to prove behavioral compatibility.

Use `update` to create or deliberately change the resolved package set and lock file. Use `install` to materialize the lock file in a clean environment; without a lock file, install must perform a fresh resolution. The lock file helps reproduce package selections but does not freeze PHP, extensions, operating-system libraries, plugins, scripts, or application configuration. Use `depends`, `prohibits`, `validate`, `check-platform-reqs`, and `diagnose` to answer distinct troubleshooting questions, then test reviewed updates in the target runtime.

## References

- [Composer: Versions and constraints](https://getcomposer.org/doc/articles/versions.md)
- [Composer: Repositories](https://getcomposer.org/doc/05-repositories.md)
- [Composer: Platform dependencies](https://getcomposer.org/doc/articles/composer-platform-dependencies.md)
- [Composer schema: Package links](https://getcomposer.org/doc/04-schema.md#package-links)
- [Composer schema: Minimum stability](https://getcomposer.org/doc/04-schema.md#minimum-stability)
- [Composer configuration: platform](https://getcomposer.org/doc/06-config.md#platform)
- [Composer configuration: dependency policies](https://getcomposer.org/doc/06-config.md#policy)
- [Composer changelog](https://github.com/composer/composer/blob/main/CHANGELOG.md)
- [Composer CLI: install and update](https://getcomposer.org/doc/03-cli.md#install-i)
- [Composer CLI: depends / why and prohibits / why-not](https://getcomposer.org/doc/03-cli.md#depends-why)
- [Composer troubleshooting](https://getcomposer.org/doc/articles/troubleshooting.md)
- [Composer source: Installer](https://github.com/composer/composer/blob/main/src/Composer/Installer.php)
- [Chapter 92 — Composer](092-composer.md)
- [Chapter 94 — composer.json](094-composer-json.md)
- [Chapter 95 — composer.lock](095-composer-lock.md)
- [Chapter 96 — Semantic Versioning](096-semantic-versioning.md)
