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

**Trust does not scale with reading.** A Xulo program's *structural*
properties — its types, its mutability defaults, its exhaustive matches — are
enforced by the language, so a reader who skips code still inherits them.
*Intent* is never fully enforceable, and this specification does not pretend
otherwise: intent is examined at boundaries, in signatures, and in the
rendered result. The goals below are the commitments that make the structural
half of that claim hold; the later chapters elaborate them one at a time.

Three depths of attention read the same `.xulo` file: the readers who verify
semantics — implementers, framework authors, reviewers of this specification;
the readers who judge interfaces — application developers and code reviewers
with a limited attention budget per line; and the readers who consume
results — designers, stakeholders, and automated agents for whom the rendered
outcome is the only contact with the program. One file must serve all three
without a translation layer. Automated readers multiply how often the first
two depths are consumed without multiplying how often intent is judged — which
is why the guarantees below live in the checker rather than in prose.
Correspondingly, Xulo is not aimed at fields whose primary readers must
verify every line — formal verification, kernel code, high-frequency trading
cores. Those fields need a language that *requires* full reading; Xulo is the
language that makes full reading optional for structural trust.

### Layer 1 — Properties are enforced

The trust substrate: what is checked here is checked for everyone, whether or
not anyone reads the code.

**Safe defaults.** `let` bindings are immutable unless declared `let mut`;
`const` names a compile-time constant; a `match` must account for every case of
the value it destructures; and `+` never mixes strings and numbers —
conversion is written `str(x)`. Programmers opt *into* mutation, conversion,
and openness, not out of them.

**A predictable evaluation model.** Subexpressions evaluate left to right,
blocks and patterns delimit scope, no value is coerced silently between
unrelated types, and the grammar is newline-insensitive with `;` an optional
statement terminator — so formatting never changes the meaning of a file.

### Layer 2 — Contracts are legible

The attention interface: this layer is what a reviewer needs to read; beyond
it, reading the code is optional.

**Local inference, explicit signatures at module boundaries.** Types are
inferred inside function bodies and at `let` bindings, so annotations appear
where they carry information. Declarations that cross a module boundary —
`pub` items and module-level `fn` signatures — carry explicit types, so the
interface of a file can be read without reading its body.

**Syntax written to be read.** Keywords are words rather than symbol chains
(`and`, `or`, `!`), mutability is spelled out (`let mut`), and the spelling of
a construct rarely changes with context. The result is legible to a human
reader and to a language model from the same text, with no dialect in which
the two differ. For an automated reader the same surface is *addressable*,
not merely legible: a rejected program names its diagnostic code, an
interface is extractable from its signatures alone, and the AST is a fixed
set of node names ([`ast.md`](ast.md)) — nothing a reviewer needs waits in
prose.

### Layer 3 — The result is the fact

The experience surface: a component's source *is* the interface it renders,
so consuming results instead of reading code is not a compromise — it is an
admitted way of working. What the language fixes is the tree and the moments
it updates; how a presentation target draws that tree belongs to the
implementation, and the result is trustworthy exactly to the degree that
target conforms to the semantics the later chapters specify.

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

**Zero-boilerplate reactive state.** `@State`, `@Store`, `@Effect`, and
`@Environment` appear at the top level of a component body and nothing else is
required: dependency tracking and re-rendering are part of the language design
rather than wiring the programmer assembles by hand.

### The parity principle

A feature that requires more reading must, in exchange, add a new automatic
guarantee — and *guarantee* here is checkable, not aspirational, with the
kind of check following the layer. In Layers 1 and 2 it is compile-time: a
new or extended diagnostic (a code in
[`type-system/errors.md`](type-system/errors.md)) or a new statically
enforced property, which the checker enforces. In Layer 3 it is normative
runtime semantics: behavior this specification pins down, which the
implementation must reproduce. A proposal that adds syntax with neither does
not land. The principle runs both ways. Removing syntax
while keeping its checks is always admissible at the parity gate — it
reduces reading and preserves guarantees — though it remains a
major-version change ([README §2](README.md)); removing a check is not
admissible without naming what replaces it, and is likewise major.
Guarantees accumulate downward toward Layer 1; the surface vocabulary stays
stable, so a reader who engages only lightly never has to relearn the
language as it grows.

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
  mechanism, no syntax extension, and no compile-time code generation;
  declaration attributes ([`types/traits.md`](types/traits.md)) are a closed
  registry the checker interprets, never an extension point.
- **No test framework.** The language contributes exactly two things to
  testing — the `#Test` form and the `assert` intrinsic
  ([`testing.md`](testing.md)) — so that tests are checked and discoverable
  from the source itself; how tests are organized, selected, isolated, and
  reported remains a tooling concern, and no framework construct enters the
  language.
- **No operator overloading.** Every operator has fixed operand rules, a fixed
  result type, and a fixed precedence
  ([`expressions/operators.md`](expressions/operators.md)); no `trait` or
  `impl` may redefine `+`, `==`, or any other operator for a user type.
  Structural equality is therefore the only equality a user type has.
- **No foreign-function interface contract.** The language defines no `extern`
  form, no linkage or ABI annotation, and no calling convention for code
  written outside it. The fixed-bit types exist so that a value can be laid out
  compatibly with a C ABI; binding Xulo to another language's code is a library
  and host concern.
- **No documentation comments.** A comment carries no meaning to the language:
  [`lexical-structure.md`](lexical-structure.md) defines only `//` and
  `/* … */` and states that there is no documentation-comment syntax.
  Documentation tooling may read them, but the language does not.
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
| Pattern matching | `match` over literals, types, `_`, bindings, deconstruction, and ranges, with guards and `\|` alternatives, checked for exhaustiveness | No pattern matching; `switch` plus type-narrowing checks | `match` over algebraic data types, checked for exhaustiveness |
| Module system | A package is a declared tree of modules; `pub` marks exports and member visibility; `use m::x` names a member | ESM `import`/`export` declarations; packages come from a manifest | A file is a module; `pub` controls visibility; packages are toolchain-level |

Against JavaScript and TypeScript, Xulo keeps the surface that aids reading —
`:` type annotations, generics, unions, `use`-based module names — and removes the dynamic
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
layers rather than instruction sets. Tuples stay deliberately small — at
least two elements, positional access `p.0`, destructured with `let` — so
named multi-value results still reach for `struct` or `enum`, and
trait dispatch is written explicitly, as `Trait.method(receiver)`, rather
than being resolved implicitly through a receiver's type.

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
