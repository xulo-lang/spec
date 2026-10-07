# Xulo Language Specification

**Status:** Draft
**Version:** 0.1.0
**Date:** 2026-10-06
**Language:** English

This repository contains the normative specification of the Xulo programming
language: a UI-first language in which `.xulo` source files describe both
program logic and the rendered interface.

## 1. Purpose and Scope

This specification defines the *design* of the Xulo language:

- the lexical structure of source files,
- the name, type, expression, and statement layers,
- the function, component, and module layers,
- the type-checking rules and diagnostics,
- the built-in intrinsics provided by the compiler,
- the memory, concurrency, and error-handling models,
- a complete formal grammar.

It is written for language implementers, tool authors (compilers, linters,
language servers, formatters), and advanced users who need a precise account of
what is well-formed and what it means.

### 1.1 Out of scope

The following are deliberately **not** specified here:

| Excluded | Where it is documented |
|----------|------------------------|
| The standard library (`std/*`) and library-level APIs (JSON, regular expressions, I/O) | [xulo-website reference](https://xulo.org) |
| Tutorials, guides, and getting-started material | [xulo-website](https://xulo.org), `learn/book` in the implementation repository |
| The compiler/interpreter implementation, CLI, LSP, and rendering back ends | implementation repository (`xulo`) |
| Package registry protocol and tooling | xulo-website ecosystem documentation |

Anything not covered by this specification is, by definition, outside the
language as specified.

## 2. Status and Versioning

This document is a **draft**. Chapters are marked by their presence in the
document map below; a chapter that exists is intended to be complete and
internally consistent, but the specification as a whole has not yet been
frozen.

- **Minor version** increments add or clarify content without removing
  features.
- **Major version** increments may remove or rework constructs.
- Proposed changes are tracked under [`changes/`](changes/README.md) using the
  OpenSpec-style workflow described there; accepted changes are folded into the
  main chapters and archived.

## 3. Normative Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in RFC 2119 and RFC 8174 when, and
only when, they appear in all capitals.

Additional conventions:

1. **Code examples are non-normative.** They illustrate rules; where an example
   conflicts with prose, the prose wins.
2. **Xulo source** is shown in fenced blocks tagged `xulo`. Grammar fragments
   are shown in `text` blocks using EBNF (ISO/IEC 14977) as defined in
   [`grammar.md`](grammar.md).
3. **File extension.** Xulo source files use the extension `.xulo` and MUST be
   encoded in UTF-8.
4. **Grammar is authoritative for syntax; prose is authoritative for meaning.**
   Where the two disagree, report an issue — do not pick a winner silently.
5. **Type notation.** `T?` denotes the optional type, `T | U` a union,
   `T & U` an intersection, `fn(A): R` a function type, and `Task<T>` the type
   of an asynchronous computation.

### 3.1 Normative references

The following external documents are referred to normatively; each is cited
at the point where it is used, and only for that use.

- **RFC 2119**, *Key words for use in RFCs to Indicate Requirement Levels*,
  and **RFC 8174**, *Ambiguity of Uppercase vs Lowercase in RFC 2119 Key
  Words* — the requirement key words defined in § 3.
- **ISO/IEC 14977:1996**, *Extended Backus-Naur Form* — the grammar notation
  of [`grammar.md`](grammar.md).
- **IEEE Std 754-2019**, *IEEE Standard for Floating-Point Arithmetic* — the
  semantics of `float`, `f32`, and `f64`
  ([`types/primitive-types.md`](types/primitive-types.md)).
- **RFC 3629**, *UTF-8, a transformation format of ISO 10646* — the encoding
  of `.xulo` source files (convention 3 above).

Informative, not normative: the OpenSpec-style change workflow named in § 2
and the xulo-website documents named in § 6.

## 4. Document Map

| Document | Contents |
|----------|----------|
| [`introduction.md`](introduction.md) | Language overview, design goals, non-goals, comparison with JS/TS/MoonBit |
| [`lexical-structure.md`](lexical-structure.md) | Source encoding, whitespace, comments, tokens, reserved words |
| [`names.md`](names.md) | Identifiers, naming conventions, scoping, shadowing |
| [`types/`](types/README.md) | The type constructors: primitives, composites, function types, generics, enums, traits, type relations |
| [`expressions/`](expressions/README.md) | Expression layer: literals, operators, access, calls, control flow, closures, async |
| [`statements/`](statements/README.md) | Bindings, assignment, blocks, returns, expression statements |
| [`functions.md`](functions.md) | Function definitions, parameter modes, recursion, methods |
| [`components/`](components/README.md) | Components: `View`, state, effects, environment, binding, layout |
| [`modules/`](modules/README.md) | Source files, import/export, visibility |
| [`type-system/`](type-system/README.md) | Checking rules, coercions, compile-time diagnostics (`E`/`W` codes) |
| [`builtins/`](builtins/README.md) | Prelude and compiler intrinsics |
| [`memory-and-runtime.md`](memory-and-runtime.md) | Value/reference semantics, ownership, borrowing, runtime model |
| [`concurrency.md`](concurrency.md) | Tasks, `spawn`, cancellation, structured concurrency |
| [`error-handling.md`](error-handling.md) | Thrown errors: `throw`, `try`/`catch`, error types, `Result` patterns |
| [`grammar.md`](grammar.md) | Complete EBNF grammar |
| [`ast.md`](ast.md) | Abstract syntax tree node definitions and data structures |
| [`machine-readable.md`](machine-readable.md) | Downloadable grammar files for parser generators (EBNF, Pest, tree-sitter) |
| [`changes/`](changes/README.md) | Incremental change proposals (OpenSpec style) |

## 5. Terminology

Terms are used consistently across chapters; the definitions below are
normative for the whole specification.

| Term | Definition |
|------|------------|
| **binding** | An association between a name and a value or type introduced by `let`, `let mut`, `const`, a parameter, or a declaration. |
| **component** | A function whose declared return type is `View`; constructs a fragment of the UI tree. See [`components/`](components/README.md). |
| **declaration** | A top-level or member-level construct that introduces a name: `fn`, `struct`, `enum`, `trait`, `impl`, `type`, `let`, `const`. |
| **diagnostic** | A message emitted for a program the checker rejects or warns about, identified by an `E` or `W` code; a compile-time event, not a runtime one. See [`type-system/errors.md`](type-system/errors.md). |
| **error (thrown)** | A value raised by `throw` and handled by `catch`; an ordinary runtime event, neither a diagnostic nor a runtime failure. See [`error-handling.md`](error-handling.md). |
| **evaluation order** | The order in which subexpressions are evaluated; left-to-right unless stated otherwise. |
| **immutable binding** | A binding introduced by `let` or `const`; reassignment is a compile-time error. |
| **intrinsic** | A function or namespace provided directly by the compiler, available without import. See [`builtins/`](builtins/README.md). |
| **module** | A single source file together with its exported names. |
| **optional type** | `T?`, shorthand for `T | null`. |
| **pattern** | The left-hand side of a `match` arm; deconstructs a scrutinee value. |
| **runtime failure** | A condition that stops a running program — an out-of-bounds index, division by zero — and cannot be caught by `try`. See [`memory-and-runtime.md`](memory-and-runtime.md). |
| **scrutinee** | The expression evaluated by `match` and tested against patterns. |
| **task** | A unit of asynchronous computation with type `Task<T>`. See [`concurrency.md`](concurrency.md). |
| **unit** | The type of expressions that produce no meaningful value; written `unit`. |
| **View** | The type produced by a component; a value of the render tree. |
| **view function** | Synonym for *component*. |

## 6. Relationship to Other Documents

```
this specification   normative: what Xulo is and what programs mean
        ▲
        │ explains / teaches
xulo-website docs    narrative reference, tutorials, stdlib documentation
        ▲
        │ demonstrates
implementation       lexer, parser, checker, interpreter, tooling
```

- **xulo-website content** (`content/docs/en/**`) is the primary narrative
  source this specification consolidates. Where the website and this document
  differ, this document is normative for the language design.
- **The implementation repository** (`xulo`) is *non-normative*. It may lag or
  lead the specification.
- **`learn/book`** is a learning guide and is non-normative.

## 7. Contributing

Chapters are edited as plain Markdown. When changing syntax, update
[`grammar.md`](grammar.md) in the same change; when changing terminology,
update §5 above. Proposed language changes must land under
[`changes/`](changes/README.md) before they are folded into the main chapters.
