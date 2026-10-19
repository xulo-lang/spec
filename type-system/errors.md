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
  │         ^^ expected `string`, found `int`
  │
 --> app.xulo:1:10
  │
1 │ fn greet(name: string): string { "Hello, " + name }
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
| `E0201`–`E0299` | types, expressions, inference | this chapter, [`checking-rules.md`](checking-rules.md), [`../types/generics.md`](../types/generics.md) |
| `E0301`–`E0399` | patterns and `match` | [`../expressions/control-flow.md`](../expressions/control-flow.md) |
| `E0401`–`E0499` | mutability, ownership, borrowing, shared state, and spawn | [`../statements/let-and-assignment.md`](../statements/let-and-assignment.md), [`../memory-and-runtime.md`](../memory-and-runtime.md), [`../expressions/closures.md`](../expressions/closures.md), [`../concurrency.md`](../concurrency.md) |
| `E0501`–`E0599` | `async` and `await` | [`../expressions/async-expressions.md`](../expressions/async-expressions.md) |
| `E0601`–`E0699` | modules, visibility, entry point | [modules](../modules/README.md) |
| `E0701`–`E0799` | components | [`../components/README.md`](../components/README.md) |
| `E0801`–`E0899` | traits, impls, generic bounds | [`../types/traits.md`](../types/traits.md), [`../types/generics.md`](../types/generics.md) |
| `W0101`–`W0199` | warnings | this chapter |

The table above is a coarse map of the families; a family's codes may be
spread across several chapters. The `Specified in` column of the tables below
gives the precise chapter for each individual code.

Every diagnostic of the families below is reported with the category
`semantic`; the categories `lexical` and `syntax` name earlier phases of the
same pipeline. A failure of those earlier phases — a BOM where a token is
expected, a position where the lexer matches no token — is raised before the
checker runs and carries no `E` or `W` code: it is reported by position and
message alone. Every code in the tables below is therefore `semantic`.

## Message style guide

- A mismatch message reads `expected <type>, found <type>`, with both types in
  backticks: ``expected `string`, found `int` ``.
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
triggers it, the shape of its message, and the chapter that specifies the rule
behind it. Every message renders with the
layout of [Diagnostic model](#diagnostic-model) — a header
`error[<code>] <category>: <message>`, an arrow line `--> file:line:column`,
the source line, a caret line, and optional notes.

### Name and declaration errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0101` | an identifier or type name resolves to nothing; a type name where a value is required; an unknown type in a signature | `print(total)` | ``cannot find value `total` in this scope`` | [names.md](../names.md) |
| `E0102` | a field, method, or enum variant does not exist | `u.namee`, `Color::Redd` | ``no member `namee` on `User` `` | [path-and-access.md](../expressions/path-and-access.md) |
| `E0103` | two declarations of one name in one scope | two `fn area` declarations | ``the name `area` is defined multiple times`` | [names.md](../names.md) |
| `E0104` | a collision between bindings header forms create: two `use` entries, a `use` or `mod` name against a declaration, a glob overlap, a manifest path name against a local `mod` or alias | `use math::add;` then `use util::add` | `` `add` is bound twice `` | [import-export.md](../modules/import-export.md) |
| `E0105` | a type alias that expands to itself | `type A = A` | ``type alias `A` has a cyclic definition`` | [composite-types.md](../types/composite-types.md) |
| `E0106` | a module-scope declaration bears a reserved name | `fn print(…)` at file scope | ``the name `print` is reserved by the prelude`` | [prelude.md](../builtins/prelude.md) |
| `E0107` | a reference to the name of a `#Test` function — a call, a value use, a path, or any other mention | `helperCheck()` where `#Test fn helperCheck` exists | `` `helperCheck` is a test function and cannot be referenced `` | [testing.md](../testing.md) |
| `E0108` | an attribute name that is in no registry | `#Inline` | `` attribute `Inline` is not defined `` | [traits.md](../types/traits.md) |
| `E0109` | a registered attribute on a declaration the registry does not allow it on | `#Derive(ToString) fn main()` | `` attribute `Derive` is not valid on a function `` | [traits.md](../types/traits.md) |
| `E0110` | a registered attribute whose arguments do not fit the registered shape | `#Derive` | `` attribute `Derive` takes a parenthesized list of trait names `` | [traits.md](../types/traits.md) |
| `E0111` | a well-shaped argument that names a reserved, unregistered derive target | `#Derive(Display)` | `` `Display` is not a registered derive target `` | [traits.md](../types/traits.md) |
| `E0112` | `#Derive` on a type that declares type parameters | `#Derive(ToString) struct Box<T> { v: T }` | `` cannot derive `ToString` for a generic type `` | [traits.md](../types/traits.md) |

`E0111` carries the note `` = note: registered targets: `ToString` ``.

`Display` and `Debug` are reserved names in the derive-target registry of
[traits.md](../types/traits.md). Written as targets they are `E0111`; nothing
else in the language refers to them, and a future change that registers them
specifies what they mean.

The expansion of `#Derive` is an ordinary impl and its interpolations are
ordinary template positions, so it reuses the codes those constructs already
own: a collision with a hand-written `impl ToString` is `E0805`, and a field
or payload type with no string form is `E0801`.

### Type and expression errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0201` | a value is not assignable to a type that is known: an annotation or annotated initializer, a condition, a list element, an assignment, mixed fixed-bit operands, an operator applied to an `unknown` operand | `let s: string = 42` | ``expected `string`, found `int` `` | [checking-rules.md](checking-rules.md) |
| `E0202` | an argument is not assignable to its parameter | `greet(42)` | ``expected `string`, found `int` `` | [calls.md](../expressions/calls.md) |
| `E0203` | a `return` operand or a final expression differs from the declared return type | `fn f(): int { "x" }` | ``expected `int`, found `string` `` | [return-and-block.md](../statements/return-and-block.md) |
| `E0204` | a callee has no function or component type | `let n = 1` then `n()` | `` `int` is not callable `` | [calls.md](../expressions/calls.md) |
| `E0205` | a required field is absent from an initializer | `User(name: "A")` with `age` required | ``missing field `age` in initializer of `User` `` | [composite-types.md](../types/composite-types.md) |
| `E0206` | a field does not exist on the type — in a read, an initializer, or a destructuring | `u.namee`, `let { age } = u` | `` `User` has no field `age` `` | [checking-rules.md](checking-rules.md) |
| `E0207` | the number of arguments is wrong | `f(1, 2)` for `fn f(a: int)` | ``expected 1 argument, found 2`` | [calls.md](../expressions/calls.md) |
| `E0208` | an argument label is unknown, duplicated, or a positional argument follows a named one | `f(on: 1)` | ``unknown argument label `on` `` | [calls.md](../expressions/calls.md) |
| `E0209` | a literal does not fit the type it is given — its default type or a type it is coerced into | `let b: u8 = 300` | ``literal `300` does not fit in `u8` `` | [literals.md](../expressions/literals.md) |
| `E0210` | constant arithmetic divides by zero, takes a remainder by zero, or overflows its type | `const H = 1 / 0` | ``division by zero in constant expression`` | [operators.md](../expressions/operators.md) |
| `E0211` | operands, branches, or arms have no common type | `flag ? 1 : "x"` | `` `int` and `string` have no common type `` | [checking-rules.md](checking-rules.md) |
| `E0212` | a value has no string form, in `${…}` or in `str` | `` `${user}` `` without `ToString` | `` `User` has no string form; implement `ToString` `` | [traits.md](../types/traits.md) |
| `E0213` | the iterable of a `for` loop is not a supported type | `for x in 42` | `` `int` is not iterable `` | [control-flow.md](../expressions/control-flow.md) |
| `E0214` | `break`, `continue`, or `return` is written outside its context | `return` at file scope | `` `return` outside of a function `` | [return-and-block.md](../statements/return-and-block.md) |
| `E0215` | a subscript is applied to a type that is not a `List` or `Map` | `"abc"[0]` | `` `string` cannot be indexed `` | [checking-rules.md](checking-rules.md) |
| `E0216` | a type cannot be determined: an empty collection, a bare `null`, an unannotated closure, an unsolved generic argument | `let xs = []` | `type annotations needed` | [checking-rules.md](checking-rules.md) |
| `E0217` | a `?` propagation is ill-formed: its operand is neither `T?` nor `Result<T, E>`; it appears outside a function or in one with no declared return type; or the early return's value does not fit the declared return type | `fn f(): int { half(3)? }` | ``cannot propagate `?` to return type `int``` | [error-handling.md](../error-handling.md) |
| `E0218` | an expression statement in statement position has a type other than `unit` and is neither an excepted control-flow construct nor a `send` whose task is discarded | `a + b` with `a: int` | ``expected `unit`, found `int``` | [expression-statements.md](../statements/expression-statements.md) |
| `E0219` | a tuple destructuring's initializer is not a tuple type, or the name count differs from the tuple's arity | `let (a, b) = 42` | ``expected a tuple of 2 elements, found `int` `` | [composite-types.md](../types/composite-types.md) |
| `E0220` | a positional access has a non-tuple receiver, or the position is at or past the arity | `p.2` for `p: (int, int)` | `` `(int, int)` has no element `.2` `` | [path-and-access.md](../expressions/path-and-access.md) |
| `E0221` | a typed map literal repeats a type an expected `Map<K, V>` already gives, with identifier keys or no entries | `let m: Map<string, int> = Map<string, int>{}` | ``literal `Map<string, int>{ … }` repeats the expected type`` | [composite-types.md](../types/composite-types.md) |
| `E0222` | the right operand of `is` is not a testable type, or is not assignable to the left operand's type | `data is List<int>` | ``type `List<int>` is not testable`` | [control-flow.md](../expressions/control-flow.md) |
| `E0223` | one map literal writes the same key twice directly | `{ a: 1, a: 2 }` | ``duplicate key `a` in map literal`` | [composite-types.md](../types/composite-types.md) |
| `E0224` | a `Map` key type is not hashable — it is not one of `string`, `int`, `float`, `bool`, or a payload-free `enum` | `Map<List<int>, int>{}` | ``key type `List<int>` is not hashable`` | [composite-types.md](../types/composite-types.md) |
| `E0225` | a `Set` element type is not hashable — the same set of types admitted as map keys | `Set<List<int>>` | ``element type `List<int>` is not hashable`` | [composite-types.md](../types/composite-types.md) |
| `E0226` | an intersection `&` has an operand that is not a trait type | `User & Area` | ``intersection operand `User` is not a trait`` | [composite-types.md](../types/composite-types.md) |
| `E0227` | an operand of `+` is a `string` and the other is not | `"count=" + 1` | ``cannot add `int` to `string` `` | [operators.md](../expressions/operators.md) |
| `E0228` | a parameter's default value does not fit the parameter's declared type | `fn f(name: string = 42)` | ``expected `string`, found `int` `` | [calls.md](../expressions/calls.md) |
| `E0229` | `return`, `break`, `continue`, or `?` appears in a `defer` operand | `defer { return }` | ``cannot transfer control out of a `defer` operand`` | [return-and-block.md](../statements/return-and-block.md) |
| `E0230` | a type application is ill-formed: the argument count differs from the declared type-parameter count, a generic name is written without its arguments, a non-generic name is given arguments, or the head is a type parameter | `Map<string>` | `` `Map` expects 2 type arguments, found 1 `` | [generics.md](../types/generics.md) |

### Pattern errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0301` | a `match` does not cover every variant, both booleans, or every testable member of a union without a covering arm | `match c { Color::Red => 0 }` | `` `match` does not cover every variant of `Color` (missing `Blue`) `` | [control-flow.md](../expressions/control-flow.md) |
| `E0302` | an arm can never be selected | an arm after `_ =>` | ``this arm is unreachable`` | [control-flow.md](../expressions/control-flow.md) |
| `E0303` | a pattern does not fit the scrutinee's type, or names a type that cannot be tested | `Color::Red` against `int`, `List<int> xs` anywhere | ``pattern `Color::Red` does not match `int` `` | [control-flow.md](../expressions/control-flow.md) |
| `E0304` | the alternatives of an or-pattern bind different names, or a name at different types | `int x \| string y` | ``or-pattern alternatives must bind the same names at the same types`` | [control-flow.md](../expressions/control-flow.md) |

### Place and assignment errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0401` | assignment to a place not backed by a `mut` binding — including assignment through a closure's capture of a binding not declared `let mut` (a `mut` parameter or `mut self` included) | `let count = 0` then `count = 1` | ``cannot assign twice to immutable variable `count` `` | [let-and-assignment.md](../statements/let-and-assignment.md), [closures.md](../expressions/closures.md) |
| `E0402` | the assignment target is not a place | `1 = 2`, `f() = 2` | ``invalid assignment target`` | [expression-statements.md](../statements/expression-statements.md) |

### Ownership, borrowing, shared state, and spawn errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0403` | `move` is applied to something that is not a binding path, to a borrowed binding (a parameter or receiver), or outside a `let` initializer, an argument, or a `return`; a binding is used after it has been moved out | `consume(move items)` then `print(items)` | ``cannot use moved value `items` `` | [memory-and-runtime.md](../memory-and-runtime.md) |
| `E0404` | an argument passed to a `mut` parameter or `mut self` receiver is not a mutable place, two arguments of one call overlap where either is mutably borrowed, or a mutable borrow starts while an immutable borrow of the place is outstanding | `bump(n, n)` | ``cannot borrow `n` as mutable`` | [memory-and-runtime.md](../memory-and-runtime.md) |
| `E0405` | a place is used — read, written, moved, or passed as an argument — while a mutable borrow of it is outstanding from a call that has not returned, or whose task has not settled | `xs[0] = 2` after `let t = load(xs)` | ``cannot use `xs`: it is mutably borrowed`` | [memory-and-runtime.md](../memory-and-runtime.md) |
| `E0406` | the contents of a shared binding are read or written outside `lock` on that binding | `state.total_online` outside `lock state` | ``shared state `state` is accessed outside `lock` `` | [concurrency.md](../concurrency.md) |
| `E0407` | the operand of `lock` is not a shared binding, or a value that is not shared is passed to a `shared` parameter | `lock xs { … }` with an ordinary `xs` | `` `lock` requires a shared binding `` | [concurrency.md](../concurrency.md) |
| `E0408` | a `spawn` block uses an enclosing binding that was not captured with `move` or `copy` | `spawn async { print(config) }` | `` `config` is not captured by the spawned block `` | [concurrency.md](../concurrency.md) |
| `E0409` | a type given to `spawn.process` declares no `on_message` | `spawn.process Tally { total: 0 }` | ``service `Tally` declares no `on_message` `` | [concurrency.md](../concurrency.md) |
| `E0410` | `await` appears inside the body of a `lock` expression | `await fetch_count()` inside `lock state { … }` | `` `await` is not allowed inside `lock` `` | [concurrency.md](../concurrency.md) |
| `E0411` | a place is written or moved while an immutable borrow of it is outstanding from a call that has not returned, or whose task has not settled | `xs[0] = 9` after `let t = peek(xs)` | ``cannot write `xs`: it is immutably borrowed`` | [memory-and-runtime.md](../memory-and-runtime.md) |

### `async` and `await` errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0501` | `await` is written outside an `async` context: in a non-`async` function or closure body, in a component body, in an `@Effect` body, at module top level, or in a `#Test` function that is not marked `async` | `await t` in a plain `fn` | `` `await` outside of an `async` context `` | [async-expressions.md](../expressions/async-expressions.md) |
| `E0502` | the operand of `await` is not a `Task<T>` | `await 3` | ``cannot await `int`; expected `Task<T>` `` | [async-expressions.md](../expressions/async-expressions.md) |
| `E0503` | an `async` body returns a value that differs from its declared type | returning `Task<int>` from `async fn f(): int` | ``expected `int`, found `Task<int>` `` | [async-expressions.md](../expressions/async-expressions.md) |

### Module and visibility errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0601` | a path names a member the target module does not export | `use math::nope` | ``module `math` does not export `nope` `` | [import-export.md](../modules/import-export.md) |
| `E0602` | a name exists but is not visible here | `use math::helper` with private `helper` | `` `helper` is private to `math` `` | [import-export.md](../modules/import-export.md) |
| `E0604` | a `pub` signature mentions a type that is not part of the declaring module's interface — not `pub` there, not exported there by a `pub use` chain, not universally visible | `pub fn load(): Cache` | ``private type `Cache` in public interface`` | [visibility.md](../modules/visibility.md) |
| `E0605` | a header form — `mod`, `use`, `pub use` — is written outside the file's header | a `use` after a top-level `fn` | ``module and use declarations must appear at the top of the file`` | [source-files.md](../modules/source-files.md) |
| `E0606` | a type-only binding — declared by `use type`, or propagated through a `pub use` chain — is used where a value is required | `let u = User` after `use type ui::User` | `` `User` is a type-only binding and is not a value `` | [import-export.md](../modules/import-export.md) |
| `E0607` | a path head resolves to nothing: an unknown package name, a missing `mod` file, `super` above the package root | `mod shapes;` with no `shapes.xulo` | ``cannot resolve `shapes` `` | [import-export.md](../modules/import-export.md) |
| `E0608` | the package's root module declares no `main` | a root module without `main` | ``entry module does not declare `main` `` | [source-files.md](../modules/source-files.md) |
| `E0609` | `main` has parameters or any return type but `unit` or `View` | `fn main(argc: int)` | `` `main` must take no parameters `` | [source-files.md](../modules/source-files.md) |
| `E0610` | a `mod` declaration resolves to a file that is already the body of another module, or to the file that declares it | root `form.xulo` declaring `mod form;` | ``file `form.xulo` is the body of more than one module`` | [import-export.md](../modules/import-export.md) |
| `E0611` | a file-scope initializer directly reads a file-scope `let` of a module that initializes after it, or directly calls a function declared in such a module | `let snap = config::N` in `stats.xulo`, with `mod config` ordered after `mod stats` | `` `config::N` is used before `config` initializes `` | [visibility.md](../modules/visibility.md) |

`E0603` — the import-cycle error — is **retired and reserved**: with the
declaration-built tree, initialization order is structural, so no cycle can
prevent an order from existing, and the number is never reused
([visibility.md](../modules/visibility.md)).

### Component errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0701` | `@State`, `@Store`, `@Effect`, or `@Environment` is not a top-level item of a component body | `@State` in an ordinary `fn` | `` `@State` is only valid at the top level of a component body `` | [components/README.md](../components/README.md) |
| `E0702` | a child item is not a `string`, `View`, or `List<View>` — including an un-narrowed `View?`, a binding, an assignment, or a `return` | `Row { count }` | ``child of type `int` is not a `View` `` | [view-syntax.md](../components/view-syntax.md) |
| `E0703` | `$` is written outside an argument position, or on a name that is not a bindable state place of the enclosing component body | `$tally` with no `@State tally` | `` `$tally` does not name a state declaration `` | [binding.md](../components/binding.md) |
| `E0704` | a component body's trailing expression is not a `View` | `fn Panel(): View { 42 }` | ``expected `View`, found `int` `` | [view-syntax.md](../components/view-syntax.md) |
| `E0705` | a component block follows a call to a function that does not declare `View` | `greet("Ada") { … }` | `` `greet` does not return `View`, so it takes no component block `` | [checking-rules.md](checking-rules.md) |
| `E0706` | `await` in a component body or an `@Effect` body | `await load()` inside a component | `` `await` is not allowed in a component body `` | [effect.md](../components/effect.md) |
| `E0707` | a component invocation has neither an argument list nor a block | `Counter` used where a call is required | ``component invocation requires an argument list or a block`` | [view-syntax.md](../components/view-syntax.md) |
| `E0708` | the dependency list of an `@Effect` is not a list literal | `@Effect fn() { … }, count` | ``the dependency list of `@Effect` must be a list literal`` | [effect.md](../components/effect.md) |
| `E0709` | the value of an `@Effect` declaration is not a `fn(): unit` | `@Effect fn(): int { … }` | `` `@Effect` requires a function of type `fn(): unit` `` | [effect.md](../components/effect.md) |
| `E0710` | an `@Environment` declaration omits its type annotation | `@Environment let router` | `` `@Environment` requires a type annotation `` | [environment.md](../components/environment.md) |
| `E0711` | `$` is written in an argument to something that is not a component | `print($count)` | `` `$count` may only be passed to a component `` | [binding.md](../components/binding.md) |
| `E0712` | a `$` place is a field path or otherwise not a single state cell | `Input(value: $user.name)` | `` `$user.name` is not a single state binding `` | [binding.md](../components/binding.md) |

Argument failures inside a component invocation — an unknown label, too many
or too few arguments, an argument of the wrong type, an undeclared callee — are
reported with the call codes `E0202`, `E0204`, `E0207`, and `E0208`.

### Trait and generic errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0801` | a type argument satisfies no bound, or an explicit `Trait.method(…)` finds no impl | `perimeter(Triangle)` for a type without `Area` | `` `Triangle` does not implement `Area` `` | [checking-rules.md](checking-rules.md) |
| `E0802` | a trait impl omits a method of the trait | `impl Area for Rect` without `area` | `` `impl Area for Rect` is missing `area` `` | [checking-rules.md](checking-rules.md) |
| `E0803` | an impl method's signature differs from the trait's | `fn area(self, k: int)` for `fn area(self)` | ```method `area` has an incompatible signature``` | [checking-rules.md](checking-rules.md) |
| `E0804` | an impl declares a method the trait does not declare | an extra `extra` in `impl Area for Rect` | `` `impl Area for Rect` declares `extra`, which `Area` does not `` | [checking-rules.md](checking-rules.md) |
| `E0805` | two impls of one trait for one type in one module | a second `impl Area for Rect` | `` `Area` is already implemented for `Rect` `` | [checking-rules.md](checking-rules.md) |
| `E0806` | a bound does not name a visible trait | `<T: int>` | ``bound `int` is not a trait`` | [generics.md](../types/generics.md) |
| `E0807` | a type-argument list before a callee that declares no type parameters | `f<int>(1)` where `f: fn(int): int` | `` `f` accepts no type arguments `` | [generics.md](../types/generics.md) |
| `E0808` | a written type-argument list of the wrong length | `first<int, string>([1])` for `fn first<T>(…)` | ``expected 1 type argument, found 2`` | [generics.md](../types/generics.md) |
| `E0809` | a member call `value.method(…)` finds more than one trait method candidate in scope for the receiver's type | `r.grow()` with two traits in scope that both implement `grow` for `Rect` | ``multiple applicable implementations of `grow` for `Rect` `` | [traits.md](../types/traits.md) |
| `E0810` | a `trait` declaration declares a type parameter list | `trait Convert<T> { fn convert(self): T }` | ``a trait may not declare type parameters`` | [traits.md](../types/traits.md) |
| `E0811` | a bound applies type arguments to a trait | `<T: Convert<string>>` | ``bound `Convert<string>` applies type arguments; a trait has none`` | [generics.md](../types/generics.md) |

## Warnings

A warning never rejects a program.

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `W0101` | a binding is never read | `let scratch = 1` unused | ``unused variable `scratch` `` | [let-and-assignment.md](../statements/let-and-assignment.md) |
| `W0102` | a `use` entry is never used, or a glob contributes no name that is used | `use math::add` unused | ``unused `use` of `add` `` | [import-export.md](../modules/import-export.md) |
| `W0103` | code can never run | a statement after `return` | ``unreachable code`` | [expression-statements.md](../statements/expression-statements.md) |

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
  = note: declare `total` before this line, or `use` it
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
warning[W0102]: unused `use` of `add`
 --> app.xulo:1:10
  │
1 │ use math::add;
  │          ^^^ this name is never used
```

## Acceptance

A conforming implementation MUST report every error this specification names
for a program it rejects, with the code this chapter assigns to the failure,
and MUST NOT reject a program for which no rule produces a diagnostic. Which
diagnostics are reported after the first, whether codes are rendered with
color, and how diagnostics are surfaced beyond the terminal are
implementation choices.
