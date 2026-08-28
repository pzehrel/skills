# Agent-Readable Package Documentation Specification

Status: repository working specification. This document defines the profile enforced by this skill;
it does not claim to be an adopted industry standard.

The terms **MUST**, **SHOULD**, and **MAY** distinguish required, recommended, and optional behavior.

## Purpose and scope

This specification applies to published JavaScript and TypeScript library packages. It makes the
installed package a version-matched source of first-party usage information, so a developer or coding
agent can progress from types to local documentation without depending on model memory or the latest
online docs.

It does not define an agent instruction format, require automatic indexing of `node_modules`, replace
types or tests, or require a documentation website. Package documentation is reference material under
the consuming repository's instructions.

## Information model

Each layer has one primary responsibility:

| Layer | Primary responsibility | Typical content |
| --- | --- | --- |
| Public types and JSDoc or TSDoc | Precise local contract and documentation discovery | Parameters, returns, defaults, invariants, errors, lifecycle, deprecation, local docs path |
| Package-root README | Orientation and first route | Purpose, install, minimal example, support boundary, documentation index link |
| Bundled documentation index | Progressive task selection | Short links grouped by goal, concept, integration, migration, or failure mode |
| Focused bundled pages | Explanation and operational guidance | Concepts, configuration, recipes, migration, troubleshooting |
| Executable examples and tests | Demonstrated behavior | Complete call patterns, edge cases, framework integration |
| Source implementation | Final diagnostic evidence | Behavior not adequately exposed by the public contract |

Types remain the source of exact shapes. Tests remain the source of verified behavior. Documentation
explains intent, semantics, and supported usage that types alone cannot express.

## Required conformance

A conforming package satisfies every applicable requirement below.

### PD-1: Root discovery

The published package **MUST** contain a package-root README. Near its beginning, the README **MUST**:

- state what the package does;
- show or link to the shortest supported usage path; and
- link to the bundled documentation index.

The README **MAY** also link to a documentation website, but the local route must remain usable without
that website.

### PD-2: Bundled documentation index

The package **MUST** publish a Markdown documentation index and every local page required by its
links. Use `docs/README.md` by default; an established equivalent is conforming when both the root
README and the type-entry discovery hint name its exact package-relative path.

The index **MUST** route readers by intent instead of requiring all documentation to be loaded. Local
links **MUST** be relative and remain inside the packed artifact. When online navigation matters,
provide a separate authoritative web URL because registry sites may not expose arbitrary tarball
files at relative URLs.

### PD-3: Type-entry discovery

A package that publishes TypeScript declarations **MUST** preserve a package-level documentation
comment in its public declaration entry point. The comment **MUST** state that version-matched local
documentation is bundled and name the index path relative to the package root.

TSDoc projects **MUST** use `@packageDocumentation` according to their toolchain. JavaScript-only
packages without declarations **SHOULD** provide the equivalent JSDoc hint at the public source entry.

Example:

```ts
/**
 * Utilities for building protocol clients.
 *
 * Offline documentation matching this installed version starts at `docs/README.md`, relative to the
 * package root.
 *
 * @packageDocumentation
 */
```

The package build **MUST** retain this hint in the artifact that consumers and language tools read.

### PD-4: Complete public declaration and semantic coverage

Every exported function, class, constructor, value, type alias, interface, and enum in the actual
package public surface **MUST** have an authoritative JSDoc or TSDoc comment. Every public property,
method, call signature, and constructor that consumers interact with **MUST** also be documented.

Determine this surface from package export maps, public entry points, and generated declarations, not
from source-level `export` keywords alone. A barrel re-export does not need a duplicate comment when
the original declaration comment is retained. Every overload **MUST** expose an applicable comment in
generated declarations and editor help; overloads with different contracts **MUST** document those
differences explicitly.

Every public callable signature **MUST** contain all applicable documentation components:

| Component | Requirement |
| --- | --- |
| Summary | State the operation, observable behavior, and essential contract. |
| `@typeParam` | Provide one entry for every generic parameter and explain its semantic role or constraint. |
| `@param` | Provide one entry for every runtime parameter, including rest parameters; explain role, optional behavior, and defaults rather than repeating the type. |
| `@returns` | Required for every non-constructor callable that can return a value; describe the result's meaning and branches. Omit only for `void` or `never`. |
| `@throws` | Document each meaningful synchronous exception, including propagated callback failures. Cover async rejection here only when that is the project convention; otherwise state it in `@remarks`. If failure exists only in a result value, cover it under `@returns` instead. |
| Overloads | Give every visible overload a complete applicable comment; use inheritance only when the toolchain is verified to preserve the exact contract. |

This rule applies equally to functions, methods, constructors, call signatures, and function-valued
properties. Documentation elsewhere in README, topic pages, an interface, an implementation body, or
another overload does not substitute for the comment attached to the consumer-visible signature
unless verified documentation inheritance connects them.

Every generic public class, interface, and type alias **MUST** also provide one `@typeParam` entry for
each generic parameter, even when the declaration is not callable.

Example of a complete callable signature:

```ts
/**
 * Creates a codec from reversible wire-format operations.
 *
 * @typeParam T - Business value represented by the codec.
 * @param definition - Serialization and parsing operations for `T`.
 * @returns A frozen codec carrying the supplied operations.
 * @throws `TypeError` when either required operation is missing.
 */
export function defineFieldCodec<T>(definition: FieldCodecDefinition<T>): FieldCodec<T>
```

Public API documentation **MUST** explain applicable semantics that signatures cannot fully encode:

- mental model, ownership, lifecycle, and resource cleanup;
- defaults, units, mutation, side effects, timing, retries, and cancellation;
- meaningful error conditions, ordering rules, security constraints, and unsupported combinations;
- recommended patterns, limitations, deprecation replacements, and compatibility boundaries.

The package does not need a page for every category. It needs coverage for every category that can
materially change correct use of that package version. A genuinely simple declaration may use a
concise summary, but it may not be left undocumented. Comments **MUST NOT** merely repeat the signature
without explaining the declaration's role.

Non-public functions **SHOULD** have comments when names and types do not reveal their purpose,
rationale, invariants, mutation, ordering, error translation, or algorithm. Trivial wrappers,
callbacks, and obvious local helpers **SHOULD NOT** receive comments merely to satisfy a numeric goal.

### PD-5: Single authority and progressive disclosure

Each fact **MUST** have one primary documentation home. Link to that source instead of maintaining
parallel prose. Keep high-frequency orientation in README, exact symbol semantics near types, and
conditional guidance in focused topic pages.

Generated API documentation **MUST** derive from maintained source comments or another declared
source of truth; generated output must not become an independently edited contract.

### PD-6: Version alignment

Code and affected documentation **MUST** change together. Documentation **MUST NOT** describe planned,
removed, or latest-online behavior as if it exists in the packed version.

When a release deprecates or breaks existing usage, the package **MUST** document the supported
replacement and migration path. Old guidance may remain only when its supported-version scope is
explicit.

### PD-7: Artifact verification

The maintainer **MUST** inspect the actual package file list with `npm pack --dry-run` or an equivalent
package-manager command before release. Repository presence alone does not prove publication.

The inspection **MUST** confirm that:

- README, the documentation index, linked local pages, declarations, and required examples are
  present;
- every public declaration and consumer-facing member retains an applicable documentation comment in
  declaration output, along with the local documentation hint;
- every public callable retains complete summary, type-parameter, parameter, return, failure, and
  overload documentation as applicable;
- relative links resolve within the artifact or intentionally target an authoritative URL;
- examples use only public exports and files available to consumers; and
- private notes, secrets, caches, generated site output, and unintended large assets are absent.

### PD-8: Trust boundary

Bundled documentation **MUST** be written as technical reference material, not hidden prompts or
instructions that claim authority over a consuming repository. A package-level `AGENTS.md` is not a
substitute for consumer documentation. No install script may alter a consumer's agent instructions or
configuration to force documentation discovery.

## Recommended practices

A conforming package **SHOULD** also:

- organize the documentation index around user goals rather than its source tree;
- keep topic pages focused enough for selective reading;
- provide examples that are type-checked, tested, or otherwise executable;
- use established JSDoc or TSDoc tags such as `@remarks`, `@example`, `@defaultValue`, `@throws`,
  `@deprecated`, and `@see` where they improve generated output;
- review the generated declaration surface as a documentation coverage inventory instead of relying
  on source comment counts;
- add documentation impact to public API review and completion criteria; and
- run existing JSDoc or TSDoc completeness linting when the repository provides it; and
- validate local Markdown links and code examples in continuous integration when practical.

A behavior-preserving internal refactor **SHOULD NOT** create documentation churn unless it changes a
documented mental model, extension point, contributor workflow, or architecture contract.

## Optional extensions

A package **MAY** additionally publish generated API references, source maps, a documentation website,
search indexes, or `/llms.txt` for web discovery. These extensions do not replace the required local
route and are not required for conformance.

Custom `package.json` documentation fields **MAY** be used by a known toolchain, but conformance must
not depend on fields that consumers and agents do not commonly implement.

## Suggested layout

The filenames below are defaults, not a requirement beyond the discovery rules:

```text
README.md
docs/
├── README.md
├── concepts.md
├── configuration.md
├── migration.md
├── recipes/
│   └── common-task.md
└── troubleshooting.md
examples/
└── basic.ts
```

When `package.json` uses a `files` allowlist, a typical publication rule is:

```json
{
  "files": [
    "dist",
    "docs",
    "examples"
  ]
}
```

npm includes recognized package-root README and license files specially, but that behavior does not
guarantee publication of documentation directories.

## Audit result

An audit should report each `PD-*` requirement as `pass`, `fail`, or `not applicable`, with the file or
command output that supports the result. Report recommended improvements separately so optional work
is not confused with a conformance failure.

For PD-4, the audit **MUST** include a callable coverage table with these columns: API signature,
summary, `@typeParam`, `@param`, `@returns`, `@throws` or result-failure coverage, overload coverage,
and retained in generated declarations. A missing applicable cell is a conformance failure.

## Authoritative references

- [TSDoc `@packageDocumentation`](https://tsdoc.org/pages/tags/packagedocumentation/)
- [TSDoc `@typeParam`](https://tsdoc.org/pages/tags/typeparam/)
- [TSDoc `@param`](https://tsdoc.org/pages/tags/param/)
- [TSDoc `@returns`](https://tsdoc.org/pages/tags/returns/)
- [TSDoc `@throws`](https://tsdoc.org/pages/tags/throws/)
- [npm `package.json` file inclusion rules](https://docs.npmjs.com/files/package.json/)
