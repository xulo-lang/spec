# Checking Rules

This chapter states the rules a checker derives its judgments from: what each
literal, expression, pattern, statement, and declaration must satisfy, and how
failed rules become diagnostics. The relations these rules consume are defined
in [`../types/type-relations.md`](../types/type-relations.md); the diagnostics
named throughout are specified in [errors.md](errors.md).

## Notation

| Judgment | Reads as |
|----------|----------|
| `Γ ⊢ e : T` | in `Γ`, expression `e` has type `T` |
| `Γ ⊢ e ⇐ T` | in `Γ`, `e` checks against expected type `T` — the type of `e` is assignable to `T` |
| `Γ ⊢ p ⇐ T ok` | pattern `p` is compatible with scrutinee type `T` and introduces its bindings |
| `A ≼ B` | a value of type `A` is assignable to `B` (subtyping or coercion) |

A rule holds when all of its premises hold; a failing judgment produces the
diagnostic named at that rule. `⊢` never appears in source code.

## Environment

`Γ` is what checking a body may consult; it contains:

- **Value bindings** — name to type and mutability: `let`/`let mut` bindings,
  `const`s, parameters, loop variables, pattern bindings,
  component state declarations, and `self`.
- **Type bindings** — `struct`, `enum`, and `trait` declarations, type aliases,
  and type parameters with their bounds (`<T: Area>` or a `where` clause).
- **Declarations** — the signatures of every module-level `fn` and every method
  of every `impl`, whole-file and order-independent
  ([`../modules/source-files.md`](../modules/source-files.md)), plus this
  file's `pub` names, import bindings, and namespace bindings with the
  visibility decided at resolution.
- **Context flags** — whether the body is `async` or a component body, the
  enclosing function's declared return type (`return`), and whether the body is
  inside a loop (`break`). An expected type is not part of `Γ`: the
  bidirectional rules push and pop it.

## Literal typing

Each literal has a default type, which an expected type may adapt exactly as
[coercion.md](coercion.md) allows (the literal rules of
[`../expressions/literals.md`](../expressions/literals.md)):

| Literal | Default type | Checked against an expected type `T` |
|---------|--------------|--------------------------------------|
| `42`, `0xff`, `0b1010`, `0o77` | `int` | adapts to a numeric `T` containing the value; otherwise `E0209` |
| `3.14` | `float` | adapts to `float`, `f32`, `f64`, or `number` when in range |
| `"ok"`, `'ok'` | `string` | takes the matching member when `T` is a union of string-literal types |
| `` `…${e}…` `` | `string` | always `string`; each `e` must be a base type or `ToString` (`E0212`) |
| `true`, `false` | `boolean` | checks only if `boolean ≼ T` |
| `null` | `null` | checks only against an optional `T?` |
| `[e₁, …]` | `list<C>`, `C` the join of the element types | each element checks against the expected element type |
| `[]` | none | requires an expected type or an annotation; else `E0216` |
| `{ k: v, … }` | structural object type | every expected field must be present and fit (`E0205`, `E0206`) |
| `(e₁, …)` | `(T₁, …, Tₙ)` with `Tᵢ` the element types | with an expected `(U₁, …, Uₙ)` each `eᵢ` checks against `Uᵢ` and the arities must match (`E0201`) |

A negative number is not part of a literal — it is unary `-` applied to one.

## Expression rules

### Identifiers and member access

- `Γ ⊢ x : T` when `x` resolves to a binding, declaration, or type of type `T`
  in the resolution order of [`../names.md`](../names.md); an identifier that
  resolves to nothing — or a type name written where a value is required — is
  `E0101`.
- `Γ ⊢ e.m : U` when `e : T` and `T` (after alias expansion) declares member
  `m` of type `U`; unknown member → `E0102`, a non-`pub` member examined
  outside its module → `E0602`. Method resolution is inherent `impl` methods
  first, then trait methods in scope, with explicit `Trait.method(recv)` always
  available; a method used without a call has its function type
  with the receiver bound. On a type parameter, access succeeds only
  through a bound.
- `Γ ⊢ p.i : Tᵢ` when `p : (T₁, …, Tₙ)` and `i` is a decimal literal naming
  an existing position — `0 ≤ i < n`; a positional access on a non-tuple
  type or at or past the arity → `E0220`
  ([`../expressions/path-and-access.md`](../expressions/path-and-access.md)).

### Enum variant paths and subscript

`E::V` requires `E` to resolve to an `enum` and `V` to be one of its variants
(`E0102` otherwise); a payload-less variant has type `E`, and a variant with a
payload is a constructor whose call must supply exactly the declared payload
arguments (`E0202`, `E0207`, `E0208`). `::` is used only for variants, in
expressions and patterns alike.

For `a[i]`: `a : list<T>` requires `i : int` and yields `T`;
`a : map<K, V>` requires `i : K` and yields `V`; any other type — `string`
included — is `E0215`. The read adds no optional: an out-of-range index or
absent key is a runtime error. As an assignment target the subscript is a
place, so its base must be mutable and its value assignable to the element type.

### Calls

- The callee must have a function or component type; anything else is `E0204`.
  Positional arguments come first; once a named argument appears all following
  arguments MUST be named and named arguments MAY be reordered.
  Unknown or duplicate label, or a positional argument after a named one, is
  `E0208`. Every required parameter receives exactly one argument: missing,
  excess, or skipped-but-not-defaulted parameters are `E0207`; defaults are
  evaluated where omitted, and an omitted `T?` parameter binds `null`.
- Each argument must be assignable to its parameter type (`E0202`); a method
  receiver is supplied to `self` and checked the same way. A call through a
  value of type `fn(...)` accepts positional arguments only.
- Generic calls solve type arguments by unification against the argument types,
  aided by the expected type ([`../types/generics.md`](../types/generics.md)),
  and then every bound of every parameter must be satisfied (`E0801`).

### Operators

Operand and result types, in summary; precedence and associativity are in
[`../expressions/operators.md`](../expressions/operators.md).

| Operators | Operands | Result |
|-----------|----------|--------|
| `+ - * / % **`, unary `-` | numeric, per the promotion table of [`../types/primitive-types.md`](../types/primitive-types.md) | the promoted type; constant division or remainder by zero is `E0210` |
| `+` | `string` with `string`; `list` with `list` of a common element type | `string`; that element type |
| `& \| ^ ~ << >>` | integers of one and the same type (shift amount a non-negative integer) | that type |
| `< > <= >=` | both numeric (same promotion) or both `string` | `boolean` |
| `== !=` | a common type (below) | `boolean` |
| `and`, `or`, unary `!` | `boolean` | `boolean` |
| `..< ...` | one identical numeric type on both sides | `Range<T>` |
| `=` | a mutable place and a value assignable to it | `unit` |
| `?:` | `boolean` condition; branches with a common type | that common type |
| `??` | left `T?`; right `T` or `U` | `T`, or `T \| U` |
| `?` | `T?` or `Result<T, E>`, in a function with a declared return type that admits the failure value (`E0217` otherwise) | the success type `T` |
| `await` | `Task<T>`, inside an `async` body | `T` |

Mixing distinct fixed-bit types, or a fixed-bit type with `int`, `float`, or
`number`, is `E0201` unless one side is a literal that adapts; `int + float`
promotes to `float` in arithmetic only, never in assignment.

If `a : T?` and `T` has a member of type `U`, then `a?.b : U?`,
step by step through a chain; when `a` is `null` the chain yields `null`
without evaluating anything to its right. `??` requires an optional left
operand (`E0201` otherwise); there is no optional subscript.

### Common types

The **common type** of types `T₁ … Tₙ` is a type `C` such that every `Tᵢ ≼ C`,
chosen by the first applicable rule:

1. if all `Tᵢ` are equal, `C` is that type;
2. if one `Tᵢ` is assignable to every other, `C` is that `Tᵢ`;
3. if one operand is `null` and the others share a type `T`, `C` is `T?`;
4. if every `Tᵢ` is one of `int`, `float`, `number`, `C` is `number`;
5. if the context supplies an expected type `E` and every `Tᵢ ≼ E`, `C` is `E`;
6. otherwise the operands have **no common type** and the position is `E0211`.

The checker never forms a fresh union: `int` and `string` have no common type
unless the expected type already is `int | string`. The exception is the list
literal ([`../types/composite-types.md`](../types/composite-types.md)).

### `if`, `match`, `for`, and `?:`

- The condition of `if` and of `?:` must be `boolean` (`E0201`); there is no
  truthiness. With an `else` branch both branches check against their common
  type, which is the type of the `if`; without `else` the type is `unit`.
- `match` checks its scrutinee `e : T`, each pattern against `T`
  ([Pattern typing](#pattern-typing)), then the arm bodies: in expression
  position all arm bodies must have a common type (`E0211`); in statement
  position the rule below applies. In statement position the branches and arms
  of `if`, `match`, `for`, and `while` MAY be of unrelated types.
- The iterable of a `for` must be `list<T>`, `map<K, V>` (iterating keys),
  `set<T>`, or `Range<T>` — anything else is `E0213`. The loop
  variable is a fresh immutable binding of the element type, added to `Γ` for
  the body; the loop itself has type `unit`.

### Closures, `await`, templates, spread

- A closure is checked against the expected function type when there is one:
  the expected parameter types fill in omitted annotations, and the body is
  checked in `Γ` extended with the parameters. A parameter that neither an
  annotation nor the expected type determines is `E0216`; without an expected
  type, the written annotations decide the type
  ([`../expressions/closures.md`](../expressions/closures.md)). Capture is by
  reference and never crosses a `move`.
- `await e` requires an enclosing `async` body (`E0501`) and `e : Task<T>`
  (`E0502`); its type is `T`. Inside an `async` body every `return`
  operand and the trailing expression MUST have the declared type `T`, never
  `Task<T>` (`E0503`).
- Every `${e}` in a template requires `e : int | float | boolean | string` or a
  type implementing the built-in `ToString` trait; anything else is `E0212`
  ([`../expressions/literals.md`](../expressions/literals.md)). A template
  literal always has type `string`.
- Prefix `...e` is well-formed only as a list-literal element (`e` a `list`) or
  an object-literal field (`e` an object); a wrong operand type is `E0201`, and
  elsewhere `...` is not part of the expression grammar.

### Component invocation and children

A component call is a call to a function whose declared return type is `View`;
its arguments follow the call rules above, and a trailing block contributes
children instead of arguments. Each child item must be a component invocation,
control flow, a string, or an expression of type `View` or `list<View>`
(`E0702` otherwise, `View?` narrowed first). `@State`, `@Store`, `@Effect`, and
`@Environment` are legal only at the top level of a component body (`E0701`),
`$` only in a component-invocation argument on a bindable state place of the
enclosing body (`E0703`), and a trailing block on a function not declaring
`View` is `E0705` ([`../components/view-syntax.md`](../components/view-syntax.md)).

## Pattern typing

A pattern is always checked against a known scrutinee type: `Γ ⊢ p ⇐ T ok`
derives the bindings the arm's body may use.

| Pattern | Premises | Bindings |
|---------|----------|----------|
| literal | the literal's type checks against `T` (`E0303` otherwise) | none |
| `_` | valid for every `T` | none |
| binding `x` | valid for every `T` | `x : T` |
| `E::V(sp₁ … spₙ)` | `T` expands to enum `E`; `V` a variant of `E`; `n` equals its payload arity; each sub-pattern checks against its payload type | the sub-patterns' bindings |
| `Type(sp₁ … spₙ)` | `T` expands to struct `Type`; one sub-pattern per field, in declaration order | the sub-patterns' bindings |
| `a..<b`, `a...b` | `T` is numeric and both operands are literals of `T`'s type | none |

Exhaustiveness: a `match` over an `enum` MUST cover every variant — a variant
pattern, possibly nested, or a wildcard or binding arm; over `boolean`, `true`
and `false` or a wildcard or binding arm; over any other type, a wildcard or
binding arm, because literals and ranges cannot cover it. Failure is `E0301`;
an arm that can never be selected is `E0302`. OR-patterns and guards do not
exist.

## Statement rules

- **Bindings.** `let x = e` checks `e` and binds `x` to its type;
  `let x: T = e` checks `e ⇐ T` (`E0201` on failure). A destructuring binding
  checks the initializer against the shape it deconstructs: `let { f, … } = e`
  requires an object type declaring every listed field (`E0206`), and
  `let (a, …) = e` requires a tuple type whose arity equals the name count
  (`E0219`). An annotation is
  REQUIRED whenever `e` leaves the type undetermined — an empty list literal,
  `null`, a closure with no expected type. Every `let` has an initializer; a
  `const` initializer MUST be a constant expression, whose division or
  remainder by zero and out-of-range results are `E0210`.
- **Assignment.** The target MUST be a place expression (`E0402`) backed by a
  `mut` binding (`E0401`), and the value MUST be assignable to the place's type
  (`E0201`). Evaluation order — right-hand side first, then receiver and index
  subexpressions left to right, then the write — is in
  [`../statements/let-and-assignment.md`](../statements/let-and-assignment.md).
- **The statement-value rule.**

  > An expression statement that is not `unit` is an error, EXCEPT when it is a
  > control-flow construct (`if`, `match`, `for`, `while`) whose value is
  > discarded.

  Statement-position `if`/`match` may therefore have branches or arms of
  unrelated types; expression position requires a common type. A violation is
  `E0218` ([`errors.md`](errors.md)).
- **`return`.** Inside a function with declared return type `R`, `return e`
  checks `e ⇐ R` (`E0203`), and a bare `return` requires `R` to be `unit`. In
  an `async` body `e ⇐ T`, the declared type denoting `Task<T>` (`E0503`).
  `return` outside any function, and `break`/`continue` outside a loop, are
  `E0214`; the latter two take no operand.
- **`?` and `panic`.** `e?` requires `e : T?` or `e : Result<T, E>`, has the
  success type, and — on failure — performs the enclosing function's early
  return, which needs a declared return type admitting `null` or that
  `Result<T, E>`; any other operand, context, or mismatch is `E0217`.
  `panic(m)` checks `m ⇐ string` (`E0201` otherwise) and may be given any type
  the context requires — in statement position it is checked against `unit`.
  Both are specified in
  [`../error-handling.md`](../error-handling.md).

## Declaration rules

- **Inference boundary.** Every parameter of a module-level `fn` (and of a
  nested `fn`) MUST be annotated, and the return type MUST be declared whenever
  the function produces a value — `pub` functions always; an omitted return
  type means `unit` and the body is checked against `unit`. Missing annotations
  are `E0216` ([`../functions.md`](../functions.md)).
- **Bounds.** Every type argument a call site determines MUST satisfy its
  parameter's bounds even though it was inferred (`E0801`); a bound that is not
  a visible `trait` is `E0806`.
- **`impl` completeness.** A trait `impl` MUST declare every method of the
  trait (`E0802`) and no others (`E0804`), each signature — parameters, return
  type, receiver form — being mutually assignable with the declared one
  (`E0803`). Two implementations of one trait for one type are `E0805`.
- **Type aliases.** Expanding aliases MUST terminate: an alias that expands,
  directly or through a chain, to itself is `E0105`; recursion through a type
  constructor is not a cycle.
- **Duplicates and visibility.** Two declarations of one name in one scope are
  `E0103`, and a collision involving an import binding is `E0104`. A `pub`
  signature MUST NOT mention a non-`pub` type (`E0604`), every import must
  resolve to an exported name of exactly one module (`E0601`, `E0602`,
  `E0607`), and the import graph must be acyclic (`E0603`).

## Constraint generation and solving

1. **Generation.** Constraints arise from every site where types must relate:
   annotations, arguments, `return` operands, assignment targets, elements
   against an expected element type, scrutinees against patterns, arm types
   sharing a common type, and generic arguments against type parameters.
2. **Unification.** Constraints containing type variables are solved
   structurally: equal constructors decompose into their arguments, a free
   variable binds to the other side, and an already-bound variable must unify
   again or the solve fails ([`../types/generics.md`](../types/generics.md)).
   A constraint that would make a variable a proper part of itself
   (`T = list<T>`) is rejected, never accepted as a cyclic type.
3. **Discharge.** Once variables are solved, the remaining constraints are
   ordinary assignability checks between closed types, using subtyping and the
   closed coercion list of [coercion.md](coercion.md).
4. **Bidirectional propagation.** An expected type is pushed into literals,
   empty collections, closures, and generic calls before they are inferred
   bottom-up; it never overrides a type the arguments have already fixed.

An unsolved, under-constrained, or cyclic constraint set MUST be reported as a
diagnostic ([errors.md](errors.md)) — never silently defaulted.

## Acceptance

A conforming checker MAY use any algorithm, provided it accepts exactly the
programs these rules define and diagnoses exactly the rest.

## Worked examples

**Inference succeeds.**

```xulo
fn first<T>(items: list<T>): T { items[0] }

fn main() {
  let n: int = first([1, 2, 3])          // T = int
  let name = first(["a", "b"])           // T = string
  let label = n > 0 ? "positive" : "no"  // common type: string
}
```

`[1, 2, 3]` has type `list<int>`; unifying `list<T>` with `list<int>` gives
`T = int`, satisfying the parameter's empty bound set, and the annotation
receives the result. The ternary's branches are both `string`, so rule 1 gives
the common type `string`.

**An argument fails.**

```xulo
fn greet(name: string): string { "Hello, " + name }

fn main() {
  greet(42)   // error[E0202]
}
```

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
