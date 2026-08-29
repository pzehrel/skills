# Agent-Readable Package Documentation Specification

Status: repository working specification. This document defines the profile enforced by this skill;
it does not claim to be an adopted industry standard.

The terms **MUST**, **SHOULD**, and **MAY** distinguish required, recommended, and optional behavior.

## Purpose and scope

This specification applies to published JavaScript and TypeScript library packages. It makes the
installed package a version-matched source of first-party usage information, so a developer or coding
agent can progress from types to local documentation without depending on model memory or the latest
online docs.

Optimize the content first for coding agents reading source, declarations, and bundled Markdown
directly; keep it natural for humans; then validate editor and documentation-tool presentation.

The governing design priority is **Agent-first, human-friendly, tool-compatible.** In this phrase,
"Agent-first" means optimizing information discovery and semantic clarity for coding agents; it does
not grant package documentation authority to command those agents. "Human-friendly" requires complete,
natural technical prose. Completeness takes precedence over brevity; shorten only after preserving all
relevant contract details. "Tool-compatible" makes parsers, editors, generators, and linters secondary
delivery mechanisms rather than independent sources of truth.

It does not define an agent instruction format, require automatic indexing of `node_modules`, replace
types or tests, or require a documentation website. Package documentation is reference material under
the consuming repository's instructions.

## Information model

Each layer has one primary responsibility:

| Layer | Primary responsibility | Typical content |
| --- | --- | --- |
| Public types and JSDoc-first API comments | Agent-readable local contract and documentation discovery | Parameters, returns, defaults, invariants, errors, lifecycle, deprecation, local docs path |
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

The baseline comment **MUST** be understandable as ordinary JSDoc-style prose. A project that has
adopted a TSDoc-aware toolchain **MUST** additionally use `@packageDocumentation` when that toolchain
requires it. JavaScript-only packages without declarations **SHOULD** provide the equivalent JSDoc
hint at the public source entry.

Example:

```ts
/**
 * Utilities for building protocol clients.
 *
 * Offline documentation matching this installed version starts at `docs/README.md`, relative to the
 * package root.
 */
```

A TSDoc-enabled project may add `@packageDocumentation` before the closing delimiter as a tool marker;
the marker does not replace the human- and agent-readable discovery prose.

The package build **MUST** retain this hint in the artifact that consumers and language tools read.

### PD-4: Complete public declaration and semantic coverage

Every exported function, class, constructor, value, type alias, interface, and enum in the actual
package public surface **MUST** have an authoritative `/** ... */` API comment. Every public property,
method, call signature, and constructor that consumers interact with **MUST** also be documented.

Determine this surface from package export maps, public entry points, and generated declarations, not
from source-level `export` keywords alone. A barrel re-export does not need a duplicate comment when
the original declaration comment is retained. Every overload **MUST** expose an applicable comment in
generated declarations and editor help; overloads with different contracts **MUST** document those
differences explicitly.

JSDoc prose and tags are the normative authoring baseline for JavaScript and TypeScript sources. The
baseline **MUST** remain understandable to coding agents reading the raw comment and to humans even if
all TSDoc-only constructs are ignored. TypeScript syntax remains authoritative for types: comments
**MUST NOT** repeat parameter or return types in braces, and **MUST NOT** use JSDoc `@template` to
redeclare a TypeScript generic. JavaScript sources **MAY** use TypeScript-supported JSDoc type tags when
comments provide the package's type information.

TSDoc is a supplementary interoperability layer. TSDoc-only tags and structures **MAY** be used when
they add value to an explicitly adopted TSDoc-aware generator, API review tool, or linter. A repository
**MAY** require those extensions through its own profile, but they are not baseline requirements merely
because the source language is TypeScript. Required contract information **MUST NOT** exist only in a
TSDoc extension that baseline JSDoc consumers may ignore.

Every public callable signature **MUST** contain all applicable semantic documentation:

| Component | Requirement |
| --- | --- |
| Summary | State the operation, observable behavior, and essential contract. |
| `@param` | Provide one entry for every runtime parameter, including rest parameters; explain role, optional behavior, and defaults rather than repeating the type. |
| Return semantics | Use `@returns` when the result has meaning not already obvious from the API name and TypeScript return type, including ownership, identity, mutability, units, branches, caching, or result-based failure. It may be omitted for `void`, `never`, or a genuinely self-explanatory result. |
| Generic semantics | Explain a type parameter when its role, constraint, relationship, or lifetime is not obvious. Put the explanation in ordinary prose or the related `@param`/`@returns`; optionally add `@typeParam` for an adopted TSDoc profile. |
| Failure semantics | Document meaningful synchronous exceptions, asynchronous rejections, propagated callback failures, and result-based failures. Use JSDoc `@throws` for thrown exceptions and keep critical nuance understandable in ordinary prose when some consumers may ignore the tag. Do not invent failure behavior. |
| Overloads | Give every visible overload a complete applicable comment; use inheritance only when the toolchain is verified to preserve the exact contract. |

This rule applies equally to functions, methods, constructors, call signatures, and function-valued
properties. Documentation elsewhere in README, topic pages, an interface, an implementation body, or
another overload does not substitute for the comment attached to the consumer-visible signature
unless verified documentation inheritance connects them.

The primary JSDoc vocabulary is summary prose, `@param`, `@returns`, `@throws`, `@deprecated`, `@see`,
`@example`, and `{@link}`. TSDoc additions such as `@packageDocumentation`, `@typeParam`, `@remarks`,
`@defaultValue`, release-stage modifiers, and TSDoc declaration references are optional enhancements
unless an adopted repository profile requires them. Generic public classes, interfaces, and type
aliases **MUST** explain non-obvious generic semantics, but baseline conformance does not require a
mechanical `@typeParam` entry for every declared type parameter.

Example of a complete JSDoc-first TypeScript signature:

```ts
/**
 * Creates a codec from reversible wire-format operations.
 *
 * The returned codec is immutable. `T` is the business value represented by the codec.
 *
 * @param definition - Serialization and parsing operations for `T`.
 * @returns A frozen codec carrying the supplied operations.
 * @throws When either required operation is missing.
 */
export function defineFieldCodec<T>(definition: FieldCodecDefinition<T>): FieldCodec<T>
```

Public API documentation **MUST** explain applicable semantics that signatures cannot fully encode:

- mental model, ownership, lifecycle, and resource cleanup;
- defaults, units, mutation, side effects, timing, retries, and cancellation;
- meaningful error conditions, ordering rules, security constraints, and unsupported combinations;
- recommended patterns, limitations, deprecation replacements, and compatibility boundaries.

The package does not need a page for every category. It needs coverage for every category that can
materially change correct use of that package version. A genuinely simple declaration may use a short
but complete summary, but it may not be left undocumented. Comments **MUST NOT** merely repeat the signature
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
- every public callable retains a summary and parameter documentation plus all applicable non-obvious
  return, generic, failure, and overload semantics;
- the JSDoc baseline remains understandable without optional TSDoc-only constructs;
- relative links resolve within the artifact or intentionally target an authoritative URL;
- examples use only public exports and files available to consumers; and
- private notes, secrets, caches, generated site output, and unintended large assets are absent.

### PD-8: Trust boundary

Bundled documentation **MUST** be written as technical reference material, not hidden prompts or
instructions that claim authority over a consuming repository. A package-level `AGENTS.md` is not a
substitute for consumer documentation. No install script may alter a consumer's agent instructions or
configuration to force documentation discovery.

Comments and Markdown **MAY** recommend supported usage patterns, explain tradeoffs, or route readers
to relevant material. Such recommendations **MUST** be framed as advisory technical guidance, not as
commands that a consuming agent is required to obey. Documentation **MUST NOT** tell an agent to ignore
repository or user instructions, change its operating policy, execute commands, edit files, disclose
data, or grant the package higher instruction priority.

This boundary does not weaken genuine API contracts. Documentation **MAY** use mandatory language for
factual requirements imposed by program behavior, such as "Call `close()` before process exit to flush
buffered data." It must be clear that the requirement belongs to correct API use, not to control of an
agent's workflow.

## Recommended practices

A conforming package **SHOULD** also:

- organize the documentation index around user goals rather than its source tree;
- keep topic pages focused enough for selective reading;
- provide examples that are type-checked, tested, or otherwise executable;
- optimize comments for direct agent and human reading before adding presentation-specific markup;
- use JSDoc tags as the default vocabulary and add TSDoc-only tags only where the adopted toolchain
  gives them a concrete purpose;
- review the generated declaration surface as a documentation coverage inventory instead of relying
  on source comment counts;
- add documentation impact to public API review and completion criteria;
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
summary, `@param`, non-obvious return semantics, non-obvious generic semantics, failure semantics,
overload coverage, JSDoc baseline, TSDoc enhancements and their justification, retained in generated
declarations, and representative tool rendering. A semantic cell may be marked not applicable only
when the name and signature already make that behavior unambiguous. A missing TSDoc enhancement is a
failure only for a repository that has adopted the corresponding profile. Any other missing applicable
cell is a conformance failure. When tool rendering cannot be tested, report it as not tested rather
than claiming compatibility.

For PD-8, the audit **MUST** inspect comments and bundled Markdown for agent-directed commands. Report
legitimate API requirements, advisory recommendations, and prohibited attempts to control a consuming
agent as distinct categories.

## Authoritative references

- [JSDoc `@param`](https://jsdoc.app/tags-param)
- [JSDoc `@returns`](https://jsdoc.app/tags-returns)
- [JSDoc `@throws`](https://jsdoc.app/tags-throws)
- [JSDoc `@see`](https://jsdoc.app/tags-see)
- [JSDoc `{@link}`](https://jsdoc.app/tags-inline-link)
- [VS Code: Editing TypeScript](https://code.visualstudio.com/docs/typescript/typescript-editing)
- [TypeScript JSDoc reference](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
- [TSDoc `@packageDocumentation`](https://tsdoc.org/pages/tags/packagedocumentation/)
- [TSDoc `@typeParam`](https://tsdoc.org/pages/tags/typeparam/)
- [TSDoc `@param`](https://tsdoc.org/pages/tags/param/)
- [TSDoc `@returns`](https://tsdoc.org/pages/tags/returns/)
- [TSDoc `@throws`](https://tsdoc.org/pages/tags/throws/)
- [npm `package.json` file inclusion rules](https://docs.npmjs.com/files/package.json/)
