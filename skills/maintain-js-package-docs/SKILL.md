---
name: maintain-js-package-docs
description: Apply a version-aligned documentation specification while developing public JavaScript or TypeScript packages, with editor-friendly public API comments and aligned README, bundled docs, and examples. Use for features, API changes, deprecations, migrations, documentation adoption, audits, and releases.
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
2. Inventory the actual consumer-facing API from `package.json` exports, public entry points, and the
   generated declaration surface. Do not equate every source-level `export` with a public package
   export.
3. Ensure every public exported function, class, constructor, value, type, interface, enum, and every
   consumer-facing public member has an authoritative `/** ... */` API comment. For TypeScript, use an
   editor-friendly JSDoc-compatible subset by default and add TSDoc-only constructs only when the
   repository's documentation toolchain consumes them. Re-export barrels do not need duplicate
   comments when the original declaration comment survives in generated declarations. Document every
   overload whose contract differs or whose comment would otherwise be absent from editor help.
4. Treat a public callable comment as incomplete until it covers every runtime parameter and all
   non-obvious return, generic, failure, and overload semantics. Describe semantic roles and behavior
   instead of restating TypeScript types or adding tags with no information beyond the signature.
5. Add comments to non-public functions when their purpose, reason, invariant, mutation, ordering,
   error translation, or algorithm is not clear from names and types. Do not comment trivial wrappers,
   callbacks, or obvious local helpers merely to increase a count.
6. Preserve or add a package-level JSDoc or TSDoc comment at the public type entry point when it helps
   users discover the package purpose and bundled documentation. Follow the repository's existing
   comment standard and ensure declaration generation retains useful comments.
7. Update the smallest authoritative human-facing document. Keep README as the concise entry point;
   route detailed concepts, tasks, recipes, migrations, and troubleshooting into focused files under
   the repository's established documentation directory.
8. Update examples when the recommended call pattern changes. Prefer examples that are type-checked,
   tested, or otherwise runnable by the existing project workflow.
9. For breaking changes and deprecations, document both the replacement and the migration path. Keep
   old guidance only when supported versions still require it, and label that scope explicitly.

Avoid copying the same contract into multiple places. Let types define exact shapes, tests define
verified behavior, README provide orientation, and topic docs explain concepts and tasks. Link between
these layers instead of maintaining parallel prose.

## Keep callable documentation complete and editor-friendly

For every exported function, public method, constructor, call signature, and function-valued public
property, require an API comment attached to the declaration consumers see. Optimize the default
TypeScript profile for hover, completion, and signature help:

- a summary that states the operation and its observable contract;
- one `@param` entry for each runtime parameter, including rest parameters, with its semantic role,
  optional behavior, and default when applicable;
- `@returns` when the result has semantics not already obvious from the name and TypeScript return
  type, such as ownership, identity, mutability, units, branches, caching, or failure representation;
- generic-parameter semantics when a type parameter's role, constraint, relationship, or lifetime is
  not obvious; put this in editor-visible prose or the related `@param`/`@returns`, and use
  `@typeParam` when the repository's TSDoc pipeline supports or requires it;
- meaningful synchronous exceptions, asynchronous rejections, callback propagation, and result-based
  failures in the form the established editor and documentation toolchain renders reliably; prefer
  visible prose for critical behavior when `@throws` is not surfaced, and do not invent a failure
  contract; and
- separate complete comments for overloads unless verified `{@inheritDoc}` or tool-supported comment
  inheritance preserves the exact contract for each visible signature.

In `.ts` and `.tsx`, let TypeScript syntax carry types: do not repeat types in braces inside `@param`
or `@returns`. Do not use JSDoc `@template` to redeclare TypeScript generics. In `.js` and `.jsx`, use
the TypeScript-supported JSDoc type tags when they provide the package's type information. Prefer the
widely supported common subset—summary prose, `@param`, `@returns`, `@deprecated`, `@see`, and
`{@link}`—unless the repository has explicitly adopted a richer TSDoc or documentation-tool profile.

Function-valued properties have the same requirements as method syntax. Keep the existing API shape
unless a change is otherwise justified; attach the tags to the property comment and verify how the
project's declaration and documentation tools render them. Documentation elsewhere does not excuse a
missing or partial signature comment.

Apply the same semantic rule to generic public classes, interfaces, and type aliases: explain a type
parameter when its role is not obvious, without mechanically requiring `@typeParam` for every generic.

Before finishing, make a temporary coverage table with one row per public callable and columns for
summary, parameters, non-obvious return semantics, non-obvious generic semantics, failures, overloads,
editor rendering, and generated-declaration retention. Mark a semantic column not applicable only
when the signature and name already make it unambiguous. Do not report completion while any applicable
cell is missing.

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
Run any existing JSDoc or TSDoc completeness linter and follow the repository's configured tag
profile; do not add a new lint dependency unless the task authorizes that tooling change.
For npm package work, inspect the actual artifact with `npm pack --dry-run` or the repository's
equivalent and confirm that README, intended docs, examples, declarations, runtime files, and source
maps follow the package policy. Do not assume a file is published merely because it exists in the
repository.

Review the diff for stale names, broken relative links, duplicated authority, comments stripped from
declarations, examples that no longer type-check, and documentation claims not covered by code or
tests. Compare the generated public declaration surface against the API inventory and list any public
symbol or member with a missing or incomplete documentation comment, including missing parameter or
non-obvious return, generic, failure, or overload semantics. Verify representative comments in editor
hover or signature help when that environment is available; otherwise mark editor rendering as not
tested. Report which documentation surfaces changed, which checks ran, and any package managers,
runtimes, or agents that were not tested. For adoption or audit work, also report unmet specification
requirements separately from optional improvements.
