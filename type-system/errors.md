# Type Errors and Diagnostics

A Xulo program is checked before it runs. Everything this specification calls
an *error* is a compile-time **diagnostic**: the checker reports it, the
program is rejected, and no part of it executes. Conditions that stop a
program after it has started — an out-of-range subscript, a division by zero on
non-constant operands — are runtime failures, called panics, and are specified
with the operations that raise them
([`../memory-and-runtime.md`](../memory-and-runtime.md)).

This chapter defines the diagnostic model, the code scheme, the message
style, and the code for every failure the rest of the specification names.

## Diagnostic model

- Every diagnostic has a **severity**: `error` or `warning`. An error rejects
  the program; a warning never does, and a program with warnings runs.
- Every diagnostic carries a **code**, a one-line **message**, and at least one
  **span** — the file, line, and column of the construct at fault.
- A diagnostic MAY carry **secondary spans**, each pointing at another
  location that participates in the failure with a short label, and **notes**
  beginning with `= note:` that explain how to repair the program.
- The primary span MUST be the construct the message speaks about; a message
  MUST NOT contradict its span.
- One failure produces one diagnostic. An implementation MAY report further
  failures in the same file after it, and MUST NOT invent a diagnostic for a
  program this specification accepts.

Rendering of a diagnostic uses this layout — a header, an arrow line for each
span, the source line under it, a caret line, and optional notes:

```text
error[E0202] semantic: argument type mismatch
 --> app.xulo:4:9
  │
4 │   greet(42)
  │         ^^ expected `String`, found `Int`
  │
 --> app.xulo:1:10
  │
1 │ fn greet(name: String): String { "Hello, " + name }
  │          ---- parameter declared here
  = note: convert the value with `str(42)` before passing it
```

- The header is `error[<code>] <category>: <message>`; for a warning it is
  `warning[<code>]: <message>`, with no category word.
- An arrow line is ` --> <file>:<line>:<column>`, one space before `-->`.
- Under it comes the source line, prefixed by the line number, a space, `│`,
  and a space; the caret line repeats that prefix with the number replaced by
  spaces, then spaces up to the span's column, then one `^` per column of the
  span. Secondary spans repeat the arrow, source, and caret lines.
- Notes are `  = note: <text>`.

Codes are stable identifiers: the wording of a message MAY be improved, but a
code MUST NOT be reused for a different failure.

## Error code ranges

| Range | Family | Specified in |
|-------|--------|--------------|
| `E0101`–`E0199` | names, declarations, reserved names | [`../names.md`](../names.md), [modules](../modules/README.md) |
| `E0201`–`E0299` | types, expressions, inference | this chapter, [`checking-rules.md`](checking-rules.md) |
| `E0301`–`E0399` | patterns and `match` | [`../expressions/control-flow.md`](../expressions/control-flow.md) |
| `E0401`–`E0499` | mutability, ownership, borrowing, shared state, and spawn | [`../statements/let-and-assignment.md`](../statements/let-and-assignment.md), [`../memory-and-runtime.md`](../memory-and-runtime.md), [`../concurrency.md`](../concurrency.md) |
| `E0501`–`E0599` | `async` and `await` | [`../expressions/async-expressions.md`](../expressions/async-expressions.md) |
| `E0601`–`E0699` | modules, visibility, entry point | [modules](../modules/README.md) |
| `E0701`–`E0799` | components | [`../components/README.md`](../components/README.md) |
| `E0801`–`E0899` | traits, impls, generic bounds | [`../types/traits.md`](../types/traits.md), [`../types/generics.md`](../types/generics.md) |
| `W0101`–`W0199` | warnings | this chapter |

Every diagnostic of the families below is reported with the category
`semantic`; the categories `lexical` and `syntax` name earlier phases of the
same pipeline.

## Message style guide

- A mismatch message reads `expected <type>, found <type>`, with both types in
  backticks: ``expected `String`, found `Int` ``.
- A name appears in backticks exactly as written in the source, and a type is
  named the way its declaration names it.
- Messages are lowercase, contain no trailing period, and fit one line; the
  sentence structure is `what is wrong`, optionally followed by `where`.
- A secondary span label says why that location matters (`parameter declared
  here`, `first bound here`); a note says what to do about it, and when a fix
  is unambiguous the note states it exactly — "declare it as `let mut count =
  0`", or "convert the value with `str(42)`".

## Error categories

One subsection per category: the trigger that raises it, an example that
triggers it, and the shape of its message. Every message renders with the
layout of [Diagnostic model](#diagnostic-model) — a header
`error[<code>] <category>: <message>`, an arrow line `--> file:line:column`,
the source line, a caret line, and optional notes.

### Name and declaration errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0101` | an identifier or type name resolves to nothing; a type name where a value is required; an unknown type in a signature | `print(total)` | ``cannot find value `total` in this scope`` |
| `E0102` | a member, namespace member, or enum variant does not exist | `u.namee`, `Color::Redd` | ``no member `namee` on `User` `` |
| `E0103` | two declarations of one name in one scope | two `fn area` declarations | ``the name `area` is defined multiple times`` |
| `E0104` | a duplicate binding in which an import participates | two imports binding `add` | `` `add` is bound twice `` |
| `E0105` | a type alias that expands to itself | `type A = A` | ``type alias `A` has a cyclic definition`` |
| `E0106` | a module-scope declaration bears a reserved name | `fn print(…)` at file scope | ``the name `print` is reserved by the prelude`` |

### Type and expression errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0201` | a value is not assignable to a type that is known: an annotation or annotated initializer, a condition, a list element, an assignment, mixed fixed-bit operands, an operator applied to an `Unknown` operand | `let s: String = 42` | ``expected `String`, found `Int` `` |
| `E0202` | an argument is not assignable to its parameter | `greet(42)` | ``expected `String`, found `Int` `` |
| `E0203` | a `return` operand or a final expression differs from the declared return type | `fn f(): Int { "x" }` | ``expected `Int`, found `String` `` |
| `E0204` | a callee has no function or component type | `let n = 1` then `n()` | `` `Int` is not callable `` |
| `E0205` | a required field is absent from an initializer | `User(name: "A")` with `age` required | ``missing field `age` in initializer of `User` `` |
| `E0206` | a field does not exist on the type — in a read, an initializer, or a destructuring | `u.namee`, `let { age } = u` | `` `User` has no field `age` `` |
| `E0207` | the number of arguments is wrong | `f(1, 2)` for `fn f(a: Int)` | ``expected 1 argument, found 2`` |
| `E0208` | an argument label is unknown, duplicated, or a positional argument follows a named one | `f(on: 1)` | ``unknown argument label `on` `` |
| `E0209` | a literal does not fit the type it is given — its default type or a type it is coerced into | `let b: U8 = 300` | ``literal `300` does not fit in `U8` `` |
| `E0210` | constant arithmetic divides by zero, takes a remainder by zero, or overflows its type | `const H = 1 / 0` | ``division by zero in constant expression`` |
| `E0211` | operands, branches, or arms have no common type | `flag ? 1 : "x"` | `` `Int` and `String` have no common type `` |
| `E0212` | a value has no string form, in `${…}` or in `str` | `` `${user}` `` without `ToString` | `` `User` has no string form; implement `ToString` `` |
| `E0213` | the iterable of a `for` loop is not a supported type | `for x in 42` | `` `Int` is not iterable `` |
| `E0214` | `break`, `continue`, or `return` is written outside its context | `return` at file scope | `` `return` outside of a function `` |
| `E0215` | a subscript is applied to a type that is not a `List` or `Map` | `"abc"[0]` | `` `String` cannot be indexed `` |
| `E0216` | a type cannot be determined: an empty collection, a bare `null`, an unannotated closure, an unsolved generic argument | `let xs = []` | `type annotations needed` |
| `E0217` | a `?` propagation is ill-formed: its operand is neither `T?` nor `Result<T, E>`; it appears outside a function or in one with no declared return type; or the early return's value does not fit the declared return type | `fn f(): Int { half(3)? }` | ``cannot propagate `?` to return type `Int``` |
| `E0218` | an expression statement in statement position has a type other than `Unit` and is not an excepted control-flow construct | `a + b` with `a: Int` | ``expected `Unit`, found `Int``` |
| `E0219` | a tuple destructuring's initializer is not a tuple type, or the name count differs from the tuple's arity | `let (a, b) = 42` | ``expected a tuple of 2 elements, found `Int` `` |
| `E0220` | a positional access has a non-tuple receiver, or the position is at or past the arity | `p.2` for `p: (Int, Int)` | `` `(Int, Int)` has no element `.2` `` |
| `E0221` | a typed map literal repeats a type an expected `Map<K, V>` already gives, with identifier keys or no entries | `let m: Map<String, Int> = Map<String, Int>{}` | ``literal `Map<String, Int>{ … }` repeats the expected type`` |
| `E0222` | the right operand of `is` is not a testable type, or is not assignable to the left operand's type | `data is List<Int>` | ``type `List<Int>` is not testable`` |

### Pattern errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0301` | a `match` does not cover every variant, both booleans, or every testable member of a union without a covering arm | `match c { Color::Red => 0 }` | `` `match` does not cover every variant of `Color` (missing `Blue`) `` |
| `E0302` | an arm can never be selected | an arm after `_ =>` | ``this arm is unreachable`` |
| `E0303` | a pattern does not fit the scrutinee's type, or names a type that cannot be tested | `Color::Red` against `Int`, `List<Int> xs` anywhere | ``pattern `Color::Red` does not match `Int` `` |

### Place and assignment errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0401` | assignment to a place not backed by a `mut` binding | `let count = 0` then `count = 1` | ``cannot assign twice to immutable variable `count` `` |
| `E0402` | the assignment target is not a place | `1 = 2`, `f() = 2` | ``invalid assignment target`` |

### Ownership, borrowing, shared state, and spawn errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0403` | `move` is applied to something that is not a binding path or appears outside a `let` initializer, an argument, or a `return`; a binding is used after it has been moved out | `consume(move items)` then `print(items)` | ``cannot use moved value `items` `` |
| `E0404` | an argument passed to a `mut` parameter or `mut self` receiver is not a mutable place, or two arguments of one call overlap where either is mutably borrowed | `bump(n, n)` | ``cannot borrow `n` as mutable`` |
| `E0405` | a place is read or written while a mutable borrow of it is outstanding — an `async` call or closure body that has not settled | `xs[0] = 2` after `let t = load(xs)` | ``cannot use `xs`: it is mutably borrowed`` |
| `E0406` | the contents of a shared binding are read or written outside `lock` on that binding | `state.total_online` outside `lock state` | ``shared state `state` is accessed outside `lock` `` |
| `E0407` | the operand of `lock` is not a shared binding, or a value that is not shared is passed to a `shared` parameter | `lock xs { … }` with an ordinary `xs` | `` `lock` requires a shared binding `` |
| `E0408` | a `spawn` block uses an enclosing binding that was not captured with `move` or `copy` | `spawn async { print(config) }` | `` `config` is not captured by the spawned block `` |
| `E0409` | a type given to `spawn.process` declares no `on_message` | `spawn.process Tally { total: 0 }` | ``service `Tally` declares no `on_message` `` |

### `async` and `await` errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0501` | `await` is written outside an `async` context: in a non-`async` function or closure body, in a component body, in an `@Effect` body, or at module top level | `await t` in a plain `fn` | `` `await` outside of an `async` context `` |
| `E0502` | the operand of `await` is not a `Task<T>` | `await 3` | ``cannot await `Int`; expected `Task<T>` `` |
| `E0503` | an `async` body returns a value that differs from its declared type | returning `Task<Int>` from `async fn f(): Int` | ``expected `Int`, found `Task<Int>` `` |

### Module and visibility errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0601` | an import or re-export names a member the target does not have | `import { nope } from "math"` | ``module `math` does not export `nope` `` |
| `E0602` | a name exists but is not visible here | `math.helper()` with private `helper` | `` `helper` is private to `math` `` |
| `E0603` | the import graph contains a cycle | `a` imports `b`, `b` imports `a` | ``import cycle: `a.xulo` → `b.xulo` → `a.xulo` `` |
| `E0604` | a `pub` signature mentions a non-`pub` type | `pub fn load(): Cache` | ``private type `Cache` in public interface`` |
| `E0605` | an `import` is written outside the file header | an `import` after a `fn` | ``imports must appear at the top of the file`` |
| `E0606` | a type-only binding is used where a value is required | `let u = User` after `import type { User }` | `` `User` was imported with `import type` and is not a value `` |
| `E0607` | a specifier resolves to nothing or to more than one module | `import "nope"` | ``cannot resolve module specifier `nope` `` |
| `E0608` | the entry module declares no `main` | an entry module without `main` | ``entry module does not declare `main` `` |
| `E0609` | `main` has parameters or any return type but `Unit` or `View` | `fn main(argc: Int)` | `` `main` must take no parameters `` |

### Component errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0701` | `@State`, `@Store`, `@Effect`, or `@Environment` is not a top-level item of a component body | `@State` in an ordinary `fn` | `` `@State` is only valid at the top level of a component body `` |
| `E0702` | a child item is not a `String`, `View`, or `List<View>` — including an un-narrowed `View?`, a binding, an assignment, or a `return` | `Row { count }` | ``child of type `Int` is not a `View` `` |
| `E0703` | `$` is written outside an argument position, or on a name that is not a bindable state place of the enclosing component body | `$tally` with no `@State tally` | `` `$tally` does not name a state declaration `` |
| `E0704` | a component body's trailing expression is not a `View` | `fn Panel(): View { 42 }` | ``expected `View`, found `Int` `` |
| `E0705` | a component block follows a call to a function that does not declare `View` | `greet("Ada") { … }` | `` `greet` does not return `View`, so it takes no component block `` |
| `E0706` | `await` in a component body or an `@Effect` body | `await load()` inside a component | `` `await` is not allowed in a component body `` |
| `E0707` | a component invocation has neither an argument list nor a block | `Counter` used where a call is required | ``component invocation requires an argument list or a block`` |
| `E0708` | the dependency list of an `@Effect` is not a list literal | `@Effect fn() { … }, count` | ``the dependency list of `@Effect` must be a list literal`` |
| `E0709` | the value of an `@Effect` declaration is not a `fn(): Unit` | `@Effect 42` | `` `@Effect` requires a function of type `fn(): Unit` `` |

Argument failures inside a component invocation — an unknown label, too many
or too few arguments, an argument of the wrong type, an undeclared callee — are
reported with the call codes `E0202`, `E0204`, `E0207`, and `E0208`.

### Trait and generic errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0801` | a type argument satisfies no bound, or an explicit `Trait.method(…)` finds no impl | `perimeter(Triangle)` for a type without `Area` | `` `Triangle` does not implement `Area` `` |
| `E0802` | a trait impl omits a method of the trait | `impl Area for Rect` without `area` | `` `impl Area for Rect` is missing `area` `` |
| `E0803` | an impl method's signature differs from the trait's | `fn area(self, k: Int)` for `fn area(self)` | ```method `area` has an incompatible signature``` |
| `E0804` | an impl declares a method the trait does not declare | an extra `extra` in `impl Area for Rect` | `` `impl Area for Rect` declares `extra`, which `Area` does not `` |
| `E0805` | two impls of one trait for one type in one module | a second `impl Area for Rect` | `` `Area` is already implemented for `Rect` `` |
| `E0806` | a bound does not name a visible trait | `<T: Int>` | ``bound `Int` is not a trait`` |

## Warnings

A warning never rejects a program.

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `W0101` | a binding is never read | `let scratch = 1` unused | ``unused variable `scratch` `` |
| `W0102` | an imported name is never used | `import { add }` unused | ``unused import `add` `` |
| `W0103` | code can never run | a statement after `return` | ``unreachable code`` |

A `pub` declaration, a `const`, and a parameter are never reported as unused:
they are part of an interface or of a signature.

## Example gallery

An unknown name:

```text
error[E0101] semantic: cannot find value `total` in this scope
 --> app.xulo:3:9
  │
3 │   print(total)
  │         ^^^^^ not found in this scope
  = note: declare `total` before this line, or import it
```

An assignment to an immutable binding, with the binding's first span:

```text
error[E0401] semantic: cannot assign twice to immutable variable `count`
 --> app.xulo:3:3
  │
3 │   count = 1
  │   ^^^^^ cannot assign to `count`
  │
 --> app.xulo:2:7
  │
2 │   let count = 0
  │       ----- first bound here
  = note: declare it as `let mut count = 0` to make it assignable
```

A `match` missing a variant:

```text
error[E0301] semantic: `match` does not cover every variant of `Theme` (missing `System`)
 --> app.xulo:2:14
  │
2 │   let name = match theme {
  │              ^^^^^ this `match` is not exhaustive
  = note: add an arm for `System`, or a `_` arm
```

A warning:

```text
warning[W0102]: unused import `add`
 --> app.xulo:1:10
  │
1 │ import { add } from "math"
  │          ^^^ this name is never used
```

## Acceptance

A conforming implementation MUST report every error this specification names
for a program it rejects, with the code this chapter assigns to the failure,
and MUST NOT reject a program for which no rule produces a diagnostic. Which
diagnostics are reported after the first, whether codes are rendered with
color, and how diagnostics are surfaced beyond the terminal are
implementation choices.
