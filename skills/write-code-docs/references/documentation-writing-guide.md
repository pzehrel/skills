# Documentation Writing Guide

Read this guide when a documentation change is broad enough to need a deliberate structure or a
coverage review. It complements the short routing rules in `SKILL.md`.

## Select the localized language

English is always the first and canonical language. Before writing, infer exactly one localized language
from the highest-confidence evidence: explicit user requirements, repository instructions, existing
filename pairs, localization configuration, and nearby documents with the same audience. Do not default
to Chinese, the user's language, or the most common language. If evidence conflicts or identifies no
localized language, ask the user before creating or changing documentation. Once selected, keep the
English file and localized translation semantically aligned; preserve code, commands, identifiers, and
links exactly.

## Translation fidelity

English-first serves Agent discovery and gives reviewers a stable source of truth. It does not permit
the localized version to become an adaptation. Translate the complete English block faithfully, then
review both blocks for:

- omitted or added claims, qualifiers, examples, or constraints;
- changed negation, certainty, obligation, permission, quantity, or conditional scope;
- inconsistent terminology for the same concept; and
- altered code tokens, paths, commands, identifiers, links, or error codes.

Idiomatic grammar is welcome, but technical meaning must not be softened, strengthened, or editorialized.
When the English source is ambiguous, flag the ambiguity or ask the user instead of resolving it silently.
Human review evaluates fidelity and readability; it is not a license for an independent rewrite.

## Agent instruction documents

`AGENTS.md` is a documentation surface and may be authoritative. Write its rules as explicit,
scoped prose: state the trigger, required action, exceptions, and verification. Keep durable rules
separate from project background and temporary status. This writing guide covers clarity, semantic
coverage, and bilingual presentation; a repository's instruction-maintenance workflow remains the
authority for admitting rules and changing hierarchy or scope.

The canonical `AGENTS.md` stays in English for Agent consumption. Put the precise human-review
translation in the repository's localized counterpart, such as `AGENTS.zh-CN.md`, and link it when
useful. A localized counterpart is not automatically authoritative or discoverable; only treat it as
loaded instructions when the relevant harness documents that behavior.

## Semantic comment coverage

Check the whole relevant implementation, not only exported functions. For every consumer-visible code
element, check the declaration, schema, or behavior a reader actually sees. This includes functions,
classes, methods, constructors, fields, properties, types, enum variants, events, commands,
configuration keys, schemas, and state transitions. Write the explanatory comment or
docstring as a complete English block, then one blank line, then the complete localized-language
counterpart inside the same comment or docstring block. Start the localized block directly without a
language label such as `Chinese:` or `中文：`. Do not split the two languages across adjacent comment
blocks, and do not interleave sentences from the two languages.

For a concise field/member whose complete bilingual text fits on one line, the compact form described
below is allowed; this formatting exception does not reduce semantic coverage. Function and method
documentation should remain detailed enough to cover every signature field.

### Function-signature coverage

For every consumer-visible function, method, constructor, callback, command handler, or other
callable, document the signature's contract at the declaration. Cover each parameter individually and
the return value; a function summary, parameter name, or type annotation is not enough. Explain each
parameter's role, accepted shape or constraints, optionality/nullability, default, units,
ownership/mutability, and cancellation or timing behavior when applicable. Explain the return value's
meaning and shape, identity/ownership, synchronous or asynchronous behavior, side effects, ordering,
and failure or rejected-input behavior. Record overload-specific differences and generic relationships.

When structured tags are supported, use one unique entry per parameter, return value, and thrown error,
and put the complete English and localized descriptions together in that entry. Keep the parameter
names, tag names, and type tokens unchanged. For languages without structured tags, use the established
docstring parameter/returns/raises sections or the nearest equivalent; do not omit the contract merely
because the syntax differs.

For example, a TypeScript callable should describe every signature field rather than only its summary:

```ts
/**
 * Sends a request using the supplied retry policy.
 *
 * 使用给定的重试策略发送请求。
 *
 * @param url Request URL; must use an allowed HTTPS origin.
 *
 *   请求 URL；必须使用允许的 HTTPS 来源。
 * @param options Retry limits and delays. Omit to use the defaults.
 *
 *   重试次数和延迟配置。省略时使用默认值。
 * @returns The parsed response body; the promise rejects on transport or validation failure.
 *
 *   解析后的响应正文；传输或校验失败时 promise 会拒绝。
 */
async function sendRequest(url: string, options?: RetryOptions): Promise<unknown> {
  // ...
}
```

Do not merely repeat `url: string` or `Promise<unknown>`; the descriptions above explain constraints,
defaults, result semantics, and failure behavior that the signature cannot express.

### File-level headers

For every newly created or materially changed code-bearing file (source code, executable script,
configuration, or schema), add one concise bilingual header at the top of the file. Place it before
imports and declarations, but after syntax-required lines such as a shebang or encoding marker and
after a repository-required license header. The header should tell a maintainer:

- what responsibility the file owns and what it deliberately does not own;
- the primary inputs, outputs, integrations, or side effects when they are not obvious; and
- a stable invariant or boundary that explains why the file exists, when applicable.

Use the language's module docstring or nearest supported file-comment form. Keep the complete English
block first, one blank line, and the complete localized block in the same comment/docstring. Keep the
header short and durable: it is an orientation aid, not a changelog, symbol index, or duplicate API
reference. Refresh it when the file's responsibility or boundary changes. Do not add headers to generated
or vendor files, or where the repository convention explicitly forbids them; preserve required license,
tool, and build directives exactly.

For example, a TypeScript file can begin with a single JSDoc header before imports:

```ts
/**
 * Coordinates retry policy construction and validation for outbound requests.
 * This module does not perform requests or persist policy state.
 *
 * 负责构建和校验出站请求的重试策略。
 * 本模块不执行请求，也不持久化策略状态。
 */

import { validate } from './validation';
```

A useful comment normally has a summary and, where applicable, explains:

- the semantic role of every input, field, option, or variant;
- optionality, defaults, units, valid ranges, ownership, and mutability;
- lifecycle, ordering, side effects, timing, retries, and cancellation;
- outputs, identity, caching, resource cleanup, and result representation;
- failures, rejected combinations, security constraints, and compatibility boundaries; and
- relationships between generic values, overloads, states, or related declarations.

Review private and internal code as well. Add comments to private declarations or implementation blocks
when their purpose, invariant, mutation, ordering, error translation, compatibility, resource/lifecycle,
performance, or security constraint is not obvious from the local code. Annotate complex algorithms,
state transitions, nested branches, meaningful loop bounds or exits, data transformations, regular
expressions, non-obvious constants, synchronization or retry logic, and cleanup paths. Put the comment
next to the smallest block that needs it and explain why the code is shaped that way. Skip trivial
wrappers, direct assignments, and obvious control flow. Run a coverage pass over exported and private
symbols, followed by a complexity pass over implementation bodies.

### Field-level coverage for structured data

For a consumer-visible interface, object type, record, dataclass, struct, class property set, enum-like
object, configuration/options object, serialized payload, or schema, document every field, property,
member, or key at its declaration. A type-level summary is useful context but is not field documentation; neither
the field name nor its type alone explains the contract. For each field whose meaning is part of the
consumer contract, state the semantic role and, when applicable, optionality or nullability, default,
units, valid range, ownership/mutability, serialization or omission behavior, lifecycle, compatibility,
and relationships or constraints involving other fields. Apply the same pass recursively to nested
consumer-visible objects.

Use the language or documentation tool's closest supported field-level form: inline JSDoc/TSDoc above a
TypeScript property, an attribute/docstring on a Python dataclass, a Go/Rust struct-field comment, a
Java/Kotlin/C# property or record-component comment, or a description in OpenAPI/JSON Schema/protobuf/
SQL/configuration metadata. If a format cannot attach a description to a field, document the fields in
the nearest authoritative schema/table/reference and link or synchronize it from the declaration. Do
not skip fields merely because they look self-explanatory; omit only genuinely private trivial fields
under the repository convention.

When a field/member's complete explanation fits on one line, use the language/tool's valid compact
single-line form. Keep both language phrases in the same comment with a clear separator, for example:

```ts
interface UserOptions {
  /** Display label shown in the UI — 在界面中显示的标签 */
  label: string;
}
```

The same idea uses different syntax outside TypeScript/JavaScript:

```go
type UserOptions struct {
	// Display label shown in the UI — 在界面中显示的标签
	Label string
}
```

```rust
struct UserOptions {
    /// Display label shown in the UI — 在界面中显示的标签
    label: String,
}
```

```python
@dataclass
class UserOptions:
    label: str  # Display label shown in the UI — 在界面中显示的标签
```

Do not use `/** ... */` in a language that does not support it. Expand the comment to a multiline
bilingual block whenever constraints, defaults, relationships, or other details do not fit.

For example, each TypeScript property gets its own bilingual block rather than relying on the interface
comment:

```ts
interface RetryOptions {
  /**
   * Maximum number of attempts, including the initial call. Must be at least 1; defaults to 3.
   *
   * 最大尝试次数，包括首次调用。必须至少为 1；默认值为 3。
   */
  maxAttempts?: number;

  /**
   * Delay between attempts in milliseconds. A value of 0 retries immediately.
   *
   * 重试之间的延迟，单位为毫秒。值为 0 时立即重试。
   */
  delayMs?: number;
}
```

Use the project's established syntax. Structured tags supplied by a language or tool (such as JSDoc or
TSDoc) are delivery aids, not the source of truth. Keep structured fields unique; in each field's
description, put the English text first, then one blank line, then the localized-language text without
a language label. Ensure the prose remains complete when those tags are ignored. Do not merely restate
types or signatures.

For example, keep English prose first, followed by one blank line and the complete localized-language
prose without a language label, and keep tags structurally singular:

```ts
/**
 * Loads the configuration file and validates its schema.
 * The returned object is detached from the parser's internal state.
 *
 * 加载配置文件并校验其 schema。
 * 返回对象与解析器的内部状态分离。
 *
 * @param path Path to the configuration file.
 *
 *   配置文件路径。
 * @returns A validated configuration.
 *
 *   已校验的配置。
 */
```

## Markdown localization layout

Markdown localization uses separate files by default. Keep one prose language per file:

```text
README.md
README.<locale>.md
docs/configuration.md
docs/configuration.<locale>.md
```

Follow an established locale directory convention when one exists. The English and localized files
must have equivalent headings, tables, links, examples, code blocks, conditions, and safety boundaries.
Do not copy the code-comment layout into Markdown by alternating English and localized paragraphs in
one file. Inline bilingual Markdown is an exception that requires an explicit repository or user rule.

## Markdown structure

Choose a page by reader intent rather than by source-tree shape:

| Reader need | Best surface |
| --- | --- |
| What is this and how do I start? | README or overview |
| How does the concept work? | Focused concept page |
| How do I perform a task? | Recipe or guide |
| How do I move from an old version? | Migration page or changelog |
| Why did this fail? | Troubleshooting page |

Keep each page focused. Link to adjacent pages instead of copying a contract. Use a stable heading
hierarchy, descriptive link text, fenced code blocks with the correct language, and examples whose
output or limitations are clear. Every human-facing page, section, example explanation, and changelog
entry must have a semantically equivalent English version followed by its localized-language version.

## Evidence and review

Trace claims to implementation, types, tests, configuration, or a named external authority. Mark an
assumption as such. Review the diff for stale API names, unsupported promises, missing links, examples
that no longer run, and text that accidentally becomes an instruction to an Agent. Once the localized
language is selected, compare both files for scope and safety—not only matching headings.
