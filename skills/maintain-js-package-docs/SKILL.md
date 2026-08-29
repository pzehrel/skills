---
name: maintain-js-package-docs
description: Maintain agent-readable, JSDoc-first public API comments and version-aligned README, bundled docs, and examples while implementing or reviewing features, fixes, refactors, or releases in publishable JavaScript or TypeScript libraries and npm packages. Use proactively whenever work can affect package exports, public APIs, behavior, defaults, errors, deprecations, examples, or packaging—even when documentation is not requested; skip app-only code and private modules without a public package contract.
license: MIT
metadata:
  repository: https://github.com/pzehrel/skills
---

# Maintain JS Package Docs

Apply the agent-readable package documentation specification while developing the package. Treat
documentation as part of the public contract and complete the implementation and every affected
documentation layer in the same change. The specification makes an installed package self-describing
and version-aligned; it does not turn package documentation into agent instructions. Optimize the
content first for coding agents reading source or declarations directly, then for natural human
reading, and finally for editor and documentation-tool presentation.

Use this design priority explicitly: **Agent-first, human-friendly, tool-compatible.**

- **Agent-first:** make purpose, contracts, examples, and routes to deeper local documentation easy to
  discover and understand from source, declarations, and the packed artifact.
- **Human-friendly:** write complete, natural technical prose that developers can understand without
  knowing an agent-specific convention. Completeness takes precedence over brevity; shorten only after
  preserving every relevant contract detail.
- **Tool-compatible:** use JSDoc as the baseline and optional TSDoc or tool-specific enhancements only
  when they improve extraction, validation, or presentation without becoming the sole source of truth.

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
   consumer-facing public member has an authoritative `/** ... */` API comment. Use JSDoc prose and
   tags as the default authoring baseline. Add TSDoc-only constructs as a supplementary layer only
   when the repository's documentation toolchain consumes them, and keep required contract semantics
   understandable without those extensions. Re-export barrels do not need duplicate comments when
   the original declaration comment survives in generated declarations. Document every overload
   whose contract differs or whose comment would otherwise be absent from consumer-visible help.
4. Treat a public callable comment as incomplete until it covers every runtime parameter and all
   non-obvious return, generic, failure, and overload semantics. Describe semantic roles and behavior
   instead of restating TypeScript types or adding tags with no information beyond the signature.
5. Add comments to non-public functions when their purpose, reason, invariant, mutation, ordering,
   error translation, or algorithm is not clear from names and types. Do not comment trivial wrappers,
   callbacks, or obvious local helpers merely to increase a count.
6. Preserve or add a package-level JSDoc-style comment at the public type entry point when it helps
   agents and users discover the package purpose and bundled documentation. Add the TSDoc
   `@packageDocumentation` marker only when the adopted toolchain uses it. Follow the repository's
   existing comment standard and ensure declaration generation retains useful comments.
7. Update the appropriate authoritative human-facing document. Keep README as a self-contained
   orientation and first route; route detailed concepts, tasks, recipes, migrations, and troubleshooting
   into focused files under the repository's established documentation directory.
8. Update examples when the recommended call pattern changes. Prefer examples that are type-checked,
   tested, or otherwise runnable by the existing project workflow.
9. For breaking changes and deprecations, document both the replacement and the migration path. Keep
   old guidance only when supported versions still require it, and label that scope explicitly.

Avoid copying the same contract into multiple places. Let types define exact shapes, tests define
verified behavior, README provide orientation, and topic docs explain concepts and tasks. Link between
these layers instead of maintaining parallel prose.

## Write agent-readable, JSDoc-first API comments

For every exported function, public method, constructor, call signature, and function-valued public
property, require an API comment attached to the declaration consumers see. Make the comment
self-contained enough for a coding agent or human to understand the contract next to the signature:

- a summary that states the operation and its observable contract;
- one `@param` entry for each runtime parameter, including rest parameters, with its semantic role,
  optional behavior, and default when applicable;
- `@returns` when the result has semantics not already obvious from the name and TypeScript return
  type, such as ownership, identity, mutability, units, branches, caching, or failure representation;
- generic-parameter semantics when a type parameter's role, constraint, relationship, or lifetime is
  not obvious; put this in ordinary prose or the related `@param`/`@returns`, and optionally add
  `@typeParam` when the repository's TSDoc pipeline supports or requires it;
- meaningful synchronous exceptions, asynchronous rejections, callback propagation, and result-based
  failures; use JSDoc `@throws` for thrown exceptions and keep any critical nuance understandable in
  ordinary prose when some consumers may ignore the tag; do not invent a failure contract; and
- separate complete comments for overloads unless verified `{@inheritDoc}` or tool-supported comment
  inheritance preserves the exact contract for each visible signature.

In `.ts` and `.tsx`, let TypeScript syntax carry types: do not repeat types in braces inside `@param`
or `@returns`. Do not use JSDoc `@template` to redeclare TypeScript generics. In `.js` and `.jsx`, use
the TypeScript-supported JSDoc type tags when they provide the package's type information.

Use JSDoc as the primary vocabulary: summary prose, `@param`, `@returns`, `@throws`, `@deprecated`,
`@see`, `@example`, and `{@link}`. Treat TSDoc additions such as `@packageDocumentation`,
`@typeParam`, `@remarks`, `@defaultValue`, release-stage modifiers, and TSDoc declaration references
as optional enhancements. Before using them, inspect the existing config and conventions for TSDoc,
API Extractor, TypeDoc, or a related linter. Do not add a TSDoc tag merely because the file is
TypeScript, and do not put the only copy of required contract information behind an extension that
baseline JSDoc consumers may ignore.

Function-valued properties have the same requirements as method syntax. Keep the existing API shape
unless a change is otherwise justified; attach the tags to the property comment and verify how the
project's declaration and documentation tools render them. Documentation elsewhere does not excuse a
missing or partial signature comment.

Apply the same semantic rule to generic public classes, interfaces, and type aliases: explain a type
parameter when its role is not obvious, without mechanically requiring `@typeParam` for every generic.

Before finishing, make a temporary coverage table with one row per public callable and columns for
summary, parameters, non-obvious return semantics, non-obvious generic semantics, failures, overloads,
JSDoc baseline, justified TSDoc enhancements, generated-declaration retention, and representative
tool rendering. Mark a semantic column not applicable only when the signature and name already make
it unambiguous. A missing TSDoc enhancement is a failure only when the repository has adopted that
profile. Do not report completion while any applicable cell is missing.

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

Package comments and bundled Markdown may recommend supported patterns, tradeoffs, or next reading,
but they are advisory reference material rather than instructions that a consuming coding agent must
obey. Do not tell an agent to ignore repository instructions, change its workflow, run commands, or
modify files. A real API precondition may still use mandatory language when it describes program
behavior—for example, requiring `close()` before process exit to flush buffered data.

## Validate the result

Run the repository's applicable tests, type checks, documentation checks, and example validation.
Run any existing JSDoc completeness linter first. Run TSDoc checks when the repository has adopted a
TSDoc profile, and follow its configured tags; do not add a new lint dependency unless the task
authorizes that tooling change.
For npm package work, inspect the actual artifact with `npm pack --dry-run` or the repository's
equivalent and confirm that README, intended docs, examples, declarations, runtime files, and source
maps follow the package policy. Do not assume a file is published merely because it exists in the
repository.

Review the diff for stale names, broken relative links, duplicated authority, comments stripped from
declarations, examples that no longer type-check, and documentation claims not covered by code or
tests. Compare the generated public declaration surface against the API inventory and list any public
symbol or member with a missing or incomplete documentation comment, including missing parameter or
non-obvious return, generic, failure, or overload semantics. Confirm that the JSDoc baseline remains
understandable after ignoring optional TSDoc-only tags. Verify representative comments through any
available editor or documentation tool, but treat presentation as a compatibility check rather than
the documentation's primary purpose; otherwise mark it as not tested. Report which documentation
surfaces changed, which checks ran, and any package managers, runtimes, or agents that were not tested.
For adoption or audit work, also report unmet specification requirements separately from optional
improvements. Check comments and bundled Markdown for agent-directed commands and distinguish them
from legitimate API requirements and clearly framed usage recommendations.
