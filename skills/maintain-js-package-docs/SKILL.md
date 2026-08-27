---
name: maintain-js-package-docs
description: Apply a version-aligned documentation specification while developing public JavaScript or TypeScript packages, keeping README, bundled docs, examples, and JSDoc or TSDoc aligned with shipped code. Use for features, API changes, deprecations, migrations, documentation adoption, audits, and releases.
license: MIT
metadata:
  repository: https://github.com/pzehrel/skills
---

# Maintain JS Package Docs

Apply the agent-readable package documentation specification while developing the package. Treat
documentation as part of the public contract and complete the implementation and every affected
documentation layer in the same change. The specification makes an installed package self-describing
and version-aligned; it does not turn package documentation into agent instructions.

## Establish the documentation impact

Before editing, inspect the effective repository instructions and the package's existing sources of
truth: `package.json`, public entry points, declaration output or source types, README, documentation
index, relevant topic pages, examples, tests, and release or migration conventions. Read only the
materials relevant to the requested change.

Identify the observable change before deciding what to document. Update documentation when the work
changes any of these:

- public exports, signatures, types, defaults, or lifecycle;
- supported configuration, environment, runtime, or compatibility;
- errors, side effects, ordering requirements, performance characteristics, or security guidance;
- recommended usage, deprecation status, migration steps, or a user-visible limitation.

Do not create documentation churn for a behavior-preserving internal refactor unless it changes a
documented mental model, extension point, contributor workflow, or architecture contract.

For specification adoption or audit, a new documentation structure, a substantial public API change,
a deprecation or migration, or an npm publication review, read
[references/package-documentation-specification.md](references/package-documentation-specification.md)
before editing. For a small change in an already conforming package, follow the established routes and
load only the affected documentation.

## Implement code and documentation together

1. Implement the requested behavior and its tests. Do not document planned behavior as if it already
   exists.
2. Update the closest public type comments. Describe semantics that types cannot express, including
   defaults, invariants, side effects, failure conditions, ownership, and lifecycle. Avoid merely
   restating the signature.
3. Preserve or add a package-level JSDoc or TSDoc comment at the public type entry point when it helps
   users discover the package purpose and bundled documentation. Follow the repository's existing
   comment standard and ensure declaration generation retains useful comments.
4. Update the smallest authoritative human-facing document. Keep README as the concise entry point;
   route detailed concepts, tasks, recipes, migrations, and troubleshooting into focused files under
   the repository's established documentation directory.
5. Update examples when the recommended call pattern changes. Prefer examples that are type-checked,
   tested, or otherwise runnable by the existing project workflow.
6. For breaking changes and deprecations, document both the replacement and the migration path. Keep
   old guidance only when supported versions still require it, and label that scope explicitly.

Avoid copying the same contract into multiple places. Let types define exact shapes, tests define
verified behavior, README provide orientation, and topic docs explain concepts and tasks. Link between
these layers instead of maintaining parallel prose.

## Keep installed documentation usable

When the package is published to npm and local documentation is part of the intended developer
experience:

- keep a short documentation route near the top of the package-root README;
- make the documentation index point to focused files through relative links;
- include the required documentation and examples in the published file allowlist;
- describe local paths relative to the package root rather than assuming a `node_modules` layout;
- treat bundled docs as version-matched reference material, not as higher-priority instructions for
  the consuming repository; and
- do not rely on `llms.txt`, a custom `package.json` field, or a dependency-level `AGENTS.md` unless
  the target toolchain explicitly implements that convention.

Do not publish, change release configuration, or broaden the package contents unless the user's task
authorizes those changes. A dry-run package inspection is read-only and may be used when relevant.

## Validate the result

Run the repository's applicable tests, type checks, documentation checks, and example validation.
For npm package work, inspect the actual artifact with `npm pack --dry-run` or the repository's
equivalent and confirm that README, intended docs, examples, declarations, runtime files, and source
maps follow the package policy. Do not assume a file is published merely because it exists in the
repository.

Review the diff for stale names, broken relative links, duplicated authority, comments stripped from
declarations, examples that no longer type-check, and documentation claims not covered by code or
tests. Report which documentation surfaces changed, which checks ran, and any package managers,
runtimes, or agents that were not tested. For adoption or audit work, also report unmet specification
requirements separately from optional improvements.
