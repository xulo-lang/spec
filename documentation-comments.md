---
title: Documentation Comments (Informative)
description: A tooling convention for structuring comment text so documentation generators and IDEs share one model.
---

## Overview

This page records a convention for reading **comment text** as documentation.
It is **informative, not normative**. The language assigns comments no meaning:
[`lexical-structure.md`](lexical-structure.md) defines only `//` and
`/* … */`, comments are discarded before parsing, and
[`modules/source-files.md`](modules/source-files.md) states that they are not
part of a module's interface. Nothing on this page changes that.

If every sentence below were deleted, the language would be unchanged. What
this page adds is agreement between tools: a source of documentation comments
MAY be read by a documentation generator and by an IDE, and both consume the
same structured model, so the generated site and the editor hint never
disagree.

Throughout, "MUST", "SHOULD", and "MAY" describe tooling behavior, never
language behavior. A violation is a defect in a tool, not a language error.

## Comment forms

The forms are ordinary comments. Their text is interpreted only by the tooling
described here.

| Source | Classification |
|--------|----------------|
| `// text` | ordinary comment |
| `/* text */` | ordinary comment; does not nest |
| `/**/` | ordinary comment (empty block) |
| `/** text */` | documentation block |
| `/***/` | documentation block, body `*` |
| `/*! text */` | inner documentation block (module or file) |
| `/// text` | documentation line |
| `//// text` | ordinary comment |
| `//! text` | inner documentation line (module or file) |

Classification rules, applied per comment before any parsing:

- A line comment is a **documentation line** when it begins with exactly
  `///` and the next character is not `/`. A line beginning with `////` or
  more slashes is an ordinary comment.
- A line comment is an **inner documentation line** when it begins with
  exactly `//!`.
- A block comment is a **documentation block** when it begins with `/**` and
  the character after `/**` is not `/`. A block beginning with `/**/` is
  ordinary. A block beginning with `/*!` is an **inner documentation block**.

`//!` and `/*! … */` document the module that owns the file (or, for an inline
`mod`, the module it encloses). `///` and `/** … */` document the declaration
that follows.

> **Block forms and `*/`.** Block comments do not nest, so a documentation
> block ends at the first `*/`. A documentation block therefore MUST NOT
> contain `*/` in its body — not even inside a fenced code block. When a body
> needs to show a block comment, use `///` lines instead.

## Extraction

The tooling extracts, for each documentation comment, one `Doc` value
(defined in [The structured model](#the-structured-model)).

### Line forms

For each line in a contiguous run of `///` (or `//!`):

1. Remove the leading `///` (or `//!`).
2. If the next character is a single space, remove that one space.
3. Keep the remainder verbatim — leading indentation, backticks, `*/`, and
   all other characters are preserved.

A bare `///` is a blank documentation line. A line that should begin with
`///` in the body is written `/// /// text`: the outer prefix is removed and
`/// text` remains.

Line forms are the Markdown-complete choice. They can carry any fenced code
block, including one that contains a block comment:

````xulo
/// Example:
///
/// ```xulo
/// /* nested */
/// fn f() { … }
/// ```
pub fn demo() { … }
````

### Block forms

For a `/** … */` (or `/*! … */`) block:

1. Take the text between the opening delimiter and the first `*/`.
2. Split it into lines on LF or CRLF; the two are equivalent,
   as in [`lexical-structure.md`](lexical-structure.md).
3. Dedent: remove the longest common leading-whitespace prefix of the
   non-blank lines, preserving relative indentation so that Markdown list
   items and indented code blocks keep their meaning.
4. Optionally strip `*` decoration. If **every** non-blank line after the
   opening delimiter matches `^\s*\*`, remove that leading whitespace, one
   `*`, and at most one following space from every such line. If any
   non-blank line does not match, strip no `*` from any line. The rule is
   all-or-nothing so that a Markdown bullet such as `* item` is never eaten.

A block whose `*/` is missing is an unterminated block comment, which
[`lexical-structure.md`](lexical-structure.md) already makes a lexical error;
tooling sees no documentation block in that case.

A prose-only block needs no `*` decoration:

```xulo
/**
Adds one to the number given.

@param a The first addend.
@param b The second addend.
@returns Their sum.
*/
pub fn add(a: int, b: int): int { a + b }
```

### Markdown

The extracted text is Markdown (CommonMark). The following are preserved and
rendered by consumers:

- paragraphs, blank lines, headings, lists, tables, blockquotes, and links;
- fenced code blocks opened by a run of three or more backticks or tildes,
  with an optional language tag;
- inline code spans.

Two rules keep Markdown and the tag layer from colliding:

- **Fence awareness.** The extractor tracks fenced code blocks. A line inside
  a fenced block is never a tag, never a section heading, and never part of
  the summary.
- **Inert code.** Code in a documentation block is text. Unlike some
  ecosystems, it is **not** executed or type-checked as a test. Tests are
  `#Test` functions ([`testing.md`](testing.md)).

### Summary and body

- The **summary** is the first non-empty paragraph outside any fence and
  before any tag line or heading. It is plain prose: consumers use it for
  completion lists and must be able to truncate it to one line.
- The **body** is the remaining Markdown, with the summary paragraph and
  recognized tag lines removed. Section headings such as `# Examples` remain
  in the body.

## Tags

A **tag** is a line whose first non-space character is `@`, followed by an
ASCII identifier, outside any fenced code block. Only the tags in this table
are recognized; any other line beginning with `@` is ordinary body text. To
keep a body line that begins with a recognized tag name as text, write it as
an inline code span or inside a fenced block.

| Tag | Shape | Meaning |
|-----|-------|---------|
| `@param` | `@param name text` | Documents one declared parameter. `name` MUST match a parameter of the declaration; for a method, `self` is implicit and is not named. |
| `@returns` | `@returns text` | Documents the returned value. |
| `@deprecated` | `@deprecated text` | Marks the declaration deprecated for editors and documentation sites. The compiler does **not** read it and emits no warning. |
| `@see` | `@see path` | A cross-reference to another declaration, written with the `::` path form of [`names.md`](names.md) and [`modules/import-export.md`](modules/import-export.md), for example `app::form::validate`. |

Tags are tooling-only. Language-level deprecation is not defined here; a
compiler-enforced form would be a language change and would need its own
attribute, diagnostic, and change proposal.

### Sections

Prose that is not bound to a particular symbol is written as a Markdown
heading, following the Rust convention. These headings are RECOMMENDED, not
required:

| Heading | Use |
|---------|-----|
| `# Examples` | Usage examples, normally a `xulo`-tagged fenced block. |
| `# Panics` | The runtime failures the declaration can raise. |
| `# Errors` | The `Result::Err` cases a fallible declaration returns. |
| `# Safety` | The caller obligations of an `extern` declaration ([`ffi.md`](ffi.md)). |

A declaration MAY use `@param`/`@returns` for symbol-bound data and Markdown
sections for prose at the same time.

````xulo
/// Adds one to the number given.
///
/// @param a The first addend.
/// @param b The second addend.
/// @returns Their sum.
///
/// # Examples
///
/// ```xulo
/// let x = add(1, 2);
/// ```
///
/// # Panics
///
/// Traps if the sum overflows an `int`.
pub fn add(a: int, b: int): int { a + b }
````

## Attachment

A documentation comment attaches to a declaration by adjacency:

- A run of documentation lines, or one documentation block, attaches to the
  next declaration.
- Only whitespace and **attribute lines** (such as `#Test`) may appear between
  the documentation comment and the declaration. Attributes decorate the same
  declaration and do not break attachment.
- A blank line, an ordinary comment, or any token that is not part of the next
  declaration breaks attachment. A documentation comment that attaches to
  nothing is *orphaned*; a tool MAY ignore it or warn.
- For a file-backed module, `//!` or `/*! … */` at the top of the file
  documents that file's module. For a `mod` declaration, `///` or `/** … */`
  immediately before it documents the declared module.

Documentation attaches to the **declaring** item, not to a use site. A tool
resolves a use back to the declaration and reads the documentation there.

For `pub use`:

- If the `pub use` carries a documentation comment, that comment documents
  the re-exported name, overriding the original for that path.
- If it carries none, the documentation of the original declaration is
  forwarded.

Documentation MAY be attached to `fn`, `struct` (including fields), `enum`
(including variants), `trait` (including methods), `type`, `const`, `mod`, and
the methods of an `impl`. Private declarations MAY carry documentation; a
consumer that produces a public document filters them out.

```xulo
//! Form validation and field construction.
//!
//! @see app::form::validate

/// A single validated field.
pub struct Field {
  /// The field's display label.
  pub label: string,
}
```

## The structured model

The extraction result is a JSON document. It is a tooling artifact: the
compiler does not produce it, no AST node exists for it, and no language rule
depends on it.

`formatVersion` is an integer, bumped when the shape changes incompatibly.
Consumers MUST ignore unknown fields.

### The `Doc` value

```json
{
  "summary": "Adds one to the number given.",
  "body": "# Examples\n\n```xulo\nlet x = add(1, 2);\n```\n\n# Panics\n\nTraps if the sum overflows an `int`.",
  "tags": [
    { "name": "param", "args": ["a"], "text": "The first addend.", "span": [120, 151] },
    { "name": "param", "args": ["b"], "text": "The second addend.", "span": [160, 192] },
    { "name": "returns", "args": [], "text": "Their sum.", "span": [200, 224] }
  ],
  "span": [110, 260],
  "attached": true
}
```

| Field | Meaning |
|-------|---------|
| `summary` | The first paragraph, as defined above. |
| `body` | The remaining Markdown, without the summary paragraph and without recognized tag lines. |
| `tags` | Recognized tags in source order. `name` is the tag name without `@`; `args` holds the whitespace-separated arguments before the free text (`name` for `@param`, the path for `@see`, empty otherwise); `text` is the free text; `span` is the tag's source range. |
| `span` | The source range of the whole documentation comment, in UTF-8 byte offsets within the module's source file. Consumers that need another encoding convert from it. |
| `attached` | Whether the comment attached to a declaration. Orphaned comments have `false`. |

### The package index

A documentation generator needs the declaration tree, so the extracted values
are assembled into a package index. The IDE uses the same nodes.

```json
{
  "formatVersion": 1,
  "package": "app",
  "modules": [
    {
      "path": "app::form",
      "source": "form.xulo",
      "doc": { "summary": "Form validation and field construction.", "body": "", "tags": [], "span": [0, 52], "attached": true },
      "items": [
        {
          "name": "Field",
          "path": "app::form::Field",
          "kind": "struct",
          "visibility": "pub",
          "signature": { "text": "pub struct Field { … }", "params": [], "returns": null, "typeParams": [], "constParams": [] },
          "doc": { "summary": "A single validated field.", "body": "", "tags": [], "span": [60, 110], "attached": true },
          "members": [
            {
              "name": "label",
              "path": "app::form::Field::label",
              "kind": "field",
              "visibility": "pub",
              "signature": { "text": "label: string", "params": [], "returns": null, "typeParams": [], "constParams": [] },
              "doc": { "summary": "The field's display label.", "body": "", "tags": [], "span": [120, 170], "attached": true },
              "members": []
            }
          ]
        }
      ]
    }
  ]
}
```

An item's `kind` is one of `fn`, `struct`, `field`, `enum`, `variant`,
`trait`, `method`, `type`, `const`, or `mod`. `path` uses the `::` form and is
unique within the package; a member's path extends its owner's path. `source`
is the module's source file, and every `span` in that module is relative to
it. `signature` is the declaration as written, plus structured parameters for
functions and methods:

```json
{
  "text": "pub fn add(a: int, b: int): int",
  "params": [
    { "name": "a", "type": "int" },
    { "name": "b", "type": "int" }
  ],
  "returns": "int",
  "typeParams": [],
  "constParams": []
}
```

Two guarantees make the index usable for cross-references and stable diffs:

- **Determinism.** Modules and items appear in source order; the serialization
  is stable for the same input.
- **Locality.** Extraction and attachment use only the documentation comment,
  its neighbours, and the declaration it touches. A tool may re-extract a
  changed region of a dirty buffer without the whole program, and an IDE
  SHOULD do so.

## Consumers

A **documentation generator** walks `modules → items → members` to build
navigation, renders each `Doc` as `summary` followed by `body`, resolves
`@see` paths to site links, renders `@deprecated` as a badge, and, for a
public site, includes only items reachable through `pub` declarations and
`pub use` re-exports.

An **IDE** indexes items by `path` and uses the same nodes:

- hover shows `summary` and `body`;
- completion shows `summary` for each candidate;
- signature help uses `signature.params` together with `@param` text;
- `@deprecated` drives strikethrough and de-prioritization;
- `@see` drives navigation;
- `span` maps a hint back to the source range.

Because both consumers read the same model, a hint and a generated page cannot
tell different stories about the same declaration.

## Relationship to the language

- Comments remain meaningless to the language. There is no
  documentation-comment syntax in [`lexical-structure.md`](lexical-structure.md),
  no `DocComment` node in [`ast.md`](ast.md), and no language diagnostic about
  documentation.
- The convention adds no tokens. `///`, `//!`, `/**`, and `/*!` are all
  written with the existing `//` and `/* */` productions; the extractor
  classifies comment text above the lexer.
- Tool-level findings — an orphaned comment, a `@param` that names no
  parameter, an unresolvable `@see`, a documentation block cut short by `*/` —
  are tool warnings. They are not `E` or `W` codes and are not registered in
  [`type-system/errors.md`](type-system/errors.md).

## Borrowed from Rust, and where we differ

The convention follows Rust's comment shapes, with one deliberate difference.

| Rust | Here | Note |
|------|------|------|
| `//`, `/* … */` | same | Ordinary comments. |
| block comments nest | **they do not** | [`lexical-structure.md`](lexical-structure.md) ends a block at the first `*/`. |
| `///` outer documentation | same, first-class | Markdown-complete, including code that contains `*/`. |
| `//!` inner documentation | same, first-class | Documents the file's module. |
| `/** … */`, `/*! … */` | accepted | Prose only; the body MUST NOT contain `*/`. |
| Markdown body, `# Examples`, `# Panics` | same | Sections are prose; `@param`/`@returns` carry symbol-bound data. |
| `#[doc = "…"]` | **not adopted** | Attributes are a closed, language-level registry; documentation stays in comments. |
| doc tests | **not adopted** | Documentation code is inert; tests are `#Test` functions. |
| `#[deprecated]` | **not adopted** | `@deprecated` is tooling-only; compiler-enforced deprecation would be a separate language change. |

## Style recommendation

- Use `///` for any documentation whose body contains a fenced example with a
  block comment, and for short one-line documentation.
- `/** … */` is accepted and is a good fit for prose-only bodies; it is
  token-cheaper when written without `*` decoration. Do not use it for a body
  containing `*/`.
- Use `//!` (or `/*! … */`) for a module's own documentation, and `///` before
  a `mod` for a submodule's.
- Put a one-sentence summary first, followed by a blank documentation line.
- A generator or an AI writing documentation SHOULD prefer `///`, whose
  failure mode is a lost line rather than an unterminated block.
