# Introduction

Xulo is a UI-first, statically typed programming language in which a single
`.xulo` source file carries both the logic of a program and the interface it
presents. Types are checked before a program runs, local type inference covers
the interior of function bodies, and a function whose declared return type is
`View` is a component: it returns a value describing a fragment of the render
tree rather than a text template. Xulo is designed to run on several
presentation targets — a terminal, a native window, and a web view — from the
same source; which target a program renders to is a property of where the
program runs, not a choice of dialect.

## Design Goals

The language design commits to the goals below; the later chapters elaborate
them one at a time.

**UI-first structure.** A program is a tree of components. Structure comes
from `{ ... }` blocks and comma-separated attributes: there are no closing tags
and no angle brackets, so a source file reads as the interface it describes.
Component blocks are deliberately token-lean — the same document written with
tag pairs would carry substantially more syntax for the same tree.

**One language for logic and layout.** Layout, reactive state, side effects,
and ordinary computation share a single syntax. There is no template language,
no markup island, and no second expression language embedded in the UI; the
`if` and `for` that shape a view are the same `if` and `for` that shape a
computation.

**Syntax written to be read.** Keywords are words rather than symbol chains
(`and`, `or`, `!`), mutability is spelled out (`let mut`), and the spelling of
a construct rarely changes with context. The result is legible to a human
reader and to a language model from the same text, with no dialect in which
the two differ.

**Safe defaults.** `let` bindings are immutable unless declared `let mut`;
`const` names a compile-time constant; a `match` must account for every case of
the value it destructures; and `+` never mixes strings and numbers —
conversion is written `str(x)`. Programmers opt *into* mutation, conversion,
and openness, not out of them.

**Local inference, explicit signatures at module boundaries.** Types are
inferred inside function bodies and at `let` bindings, so annotations appear
where they carry information. Declarations that cross a module boundary —
`pub` items and module-level `fn` signatures — carry explicit types, so the
interface of a file can be read without reading its body.

**Zero-boilerplate reactive state.** `@State`, `@Store`, `@Effect`, and
`@Environment` appear at the top level of a component body and nothing else is
required: dependency tracking and re-rendering are part of the language design
rather than wiring the programmer assembles by hand.

**A predictable evaluation model.** Subexpressions evaluate left to right,
blocks and patterns delimit scope, no value is coerced silently between
unrelated types, and the grammar is newline-insensitive with `;` an optional
statement terminator — so formatting never changes the meaning of a file.

## Non-Goals

- **Not JavaScript or TypeScript source-compatible.** A `.xulo` file is not a
  JavaScript or TypeScript program, and neither of those languages accepts
  Xulo source. The syntax and semantics of Xulo are exactly those defined by
  this specification.
- **No class-based inheritance.** Reuse is composed from `struct`, `enum`,
  `trait`, and `impl`. There is no `class` declaration, no prototype chain, and
  no subtyping by inheritance.
- **No implicit coercions.** Conversions follow exactly the rules in
  [`type-system/coercion.md`](type-system/coercion.md). There is no
  string/number juggling, no truthiness, and no automatic conversion between
  unrelated types.
- **Not a replacement for a systems language.** The fixed-bit types (`i8` …
  `u64`, `f32`, `f64`) exist for ABI and FFI precision, not for competing with
  C on its own ground: memory layout, inline assembly, and bare-metal control
  are outside the language's ambitions.
- **No macro system in v1.** The language as specified has no macro
  mechanism, no syntax extension, and no compile-time code generation. `#` is
  reserved for a possible future use and is not part of the language.
- **The standard library is out of scope.** As stated in
  [README §1.1](README.md), the standard library (`std/*`) and library-level
  APIs (JSON, regular expressions, I/O) are documented outside this
  specification; anything this specification does not define is, by
  definition, outside the language as specified.

## Language Tour

The following program uses every major construct in a few lines: an `enum`
with a `match`, a function that accumulates into a `let mut` binding, and a
component with `@State` whose text is built from a template literal.

```xulo
enum Level {
  Low
  High
}

// A match must cover every variant of the scrutinee.
fn weight(level: Level): int {
  match level {
    Level::Low => 1
    Level::High => 100
  }
}

// `let mut` makes a binding assignable; `let` bindings are not.
fn sumTo(n: int): int {
  let mut acc = 0
  for i in 1...n {
    acc = acc + i
  }
  acc
}

// A component: the declared return type is View.
fn Counter(): View {
  @State let count: int = 0

  VStack(spacing: 8) {
    Text(`count = ${count}, sum = ${sumTo(count)}`)
    Button("More", onClick: fn() { count = count + 1 })
    Text(str(weight(Level::High)))
  }
}
```

- `enum Level { ... }` introduces a type; `Level::High` names a variant. `::`
  separates a variant from its enum; `.` is used only for fields, methods, and
  namespace members.
- `match level { ... }` deconstructs the scrutinee; the arms are expressions,
  and the arms together cover every variant, so no `_` arm is needed here.
- `let mut acc = 0` creates a mutable binding — `acc = acc + i` would be an
  error on a plain `let`. `1...n` is a closed range, and `for i in ...`
  iterates it.
- `@State let count: int = 0` is a state declaration: it is legal only at the
  top level of a component body, and assigning to `count` re-renders the
  component.
- `` `count = ${count}` `` is a template literal: backticks interpolate, while
  `"..."` and `'...'` never do. `str(...)` converts explicitly, because `+`
  does not mix numbers and strings.
- `fn() { count = count + 1 }` is a closure passed as an attribute of
  `Button`; `VStack(spacing: 8) { ... }` is a component block whose children
  are components.

## Comparison with JavaScript/TypeScript/MoonBit

| Aspect | Xulo | JavaScript / TypeScript | MoonBit |
|--------|------|-------------------------|---------|
| Typing | Static, with local inference and explicit signatures at module boundaries | JavaScript is dynamic; TypeScript is static with types checked and erased | Static, with local inference and compiler-checked types |
| Mutability default | `let` immutable, `let mut` mutable, `const` a compile-time constant | `let` and `var` mutable; `const` blocks reassignment only | `let` immutable, `let mut` mutable, top-level `const` |
| Components | A component is a `fn` whose return type is `View`; the tree comes from `{ }` blocks | Components are a library convention (JSX or framework APIs) | No component model in the language; UI is a library concern |
| Async model | `async fn` has evaluated type `Task<T>`; `await` unwraps it; `spawn` starts tasks | `async fn` returns a `Promise` run on the host event loop | `async fn` for asynchronous code; effects appear in signatures |
| Pattern matching | `match` over literals, `_`, bindings, deconstruction, and ranges, checked for exhaustiveness | No pattern matching; `switch` plus type-narrowing checks | `match` over algebraic data types, checked for exhaustiveness |
| Module system | A file is a module; `pub` marks exports and member visibility; `import { a } from "m"` | ESM `import`/`export` declarations; packages come from a manifest | A file is a module; `pub` controls visibility; packages are toolchain-level |

Against JavaScript and TypeScript, Xulo keeps the surface that aids reading —
`:` type annotations, generics, unions, `import` — and removes the dynamic
base: nothing converts silently, `let` is immutable by default, and the
logical connectives are the words `and` and `or` rather than chains of `&` or
`|` characters. Visibility is a single keyword, `pub`, used both for
module-level exports and for member visibility; there is no `export`
declaration, and `export` itself is a reserved word. The comparison pages of
the introduction contrast Xulo's component blocks with JSX tag pairs: the
difference is the same design goal seen from another angle — fewer tokens,
structure that states intent.

MoonBit is the closest relative in this table. Both are statically typed, both
derive `let` from an immutable-by-default tradition, both use `pub`, and both
treat `match` as a first-class expression with exhaustiveness checking. Xulo
differs in where it points: components with a declared `View` return type,
state declarations that live in the language, and targets that are presentation
layers rather than instruction sets. Xulo also has no tuples — multi-value
results use `struct`, `object`, or `enum` — and trait dispatch is written
explicitly, as `Trait.method(receiver)`, rather than being resolved implicitly
through a receiver's type.

What Xulo takes from Swift and Rust is the vocabulary of intent: `let` and
`mut`, `where` clauses for generic constraints, `move` and `copy` for
ownership, optional types written `T?`, and `if` and `match` usable as
expressions. What it leaves behind is the symbol-heavy spelling of those ideas:
borrows and modes are written with words (`p: mut T`), there is no macro
system, and there are no classes to subclass.

## Reading Order

This specification proceeds from characters to programs: lexical structure and
names, then types, expressions, statements, functions, components, and
modules, then the checking, built-in, memory, concurrency, and error models,
ending with the complete grammar. The document map, the normative conventions
(RFC 2119 keywords, EBNF notation, code-example policy), and the terminology
table used by every chapter are in [README.md](README.md).
