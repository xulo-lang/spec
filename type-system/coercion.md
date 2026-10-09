# Coercion and Implicit Conversion

Coercion is the set of implicit conversions Xulo performs. Written `A ⇝ B`, it
is one of the two ways assignability holds: `A ≼ B` exactly when `A <: B`, or
when a coercion from this chapter applies
([`../types/type-relations.md`](../types/type-relations.md)). The set is
closed, so a failed assignability check is always a genuine type error
([errors.md](errors.md)).

## Principle

Implicit conversion is **minimal and never lossy**.

- Every coercion is listed below with the exact condition under which it
  applies, and it applies nowhere else. A conversion that is not in this
  chapter never happens.
- No coercion narrows, truncates, rounds, or otherwise discards information:
  the value seen at the target type denotes the same quantity as the value
  offered at the source type. Everything lossy or semantic — turning a number
  into text, a float into an integer, a value into a truth value — is an
  explicit call to an intrinsic, specified in
  [`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md).
- Subtyping is not coercion. Wherever `A <: B` holds, nothing converts; only
  the rows of this chapter's table convert
  ([`../types/type-relations.md`](../types/type-relations.md)).
- Xulo has no cast expression. The keyword `as` renames an imported name
  ([`../lexical-structure.md`](../lexical-structure.md)) and never converts a
  value.

## Where coercions apply

- A coercion applies only at a **checking site** — a position that has an
  expected type: a binding annotation, an argument, a `return` operand, an
  assignment target, the element or field type of a literal, a `match` arm
  result, or a `?:` branch ([`checking-rules.md`](checking-rules.md)).
- Inference never guesses a conversion. Without an expected type a literal
  takes its default type — `int`, `float`, `string`, `boolean` — and the
  result is then checked against the context; an annotation is a constraint,
  not a rewrite: `let x: T = e` requires `e ≼ T` and does not retype `e`.
- The conversion is part of the program's meaning: at the site, the value has
  the target type. Nothing converts later, and nothing converts again.

## The coercions that exist

This table is the complete set. The *kind* column says which relation carries
the row: a **coercion** converts the value, **subtyping** relates two types
without converting anything, and a **default** is simply the literal's own
type.

| From | To | Kind | Condition | On failure |
|------|----|------|-----------|------------|
| an integer literal | `int` | default | always | — |
| an integer literal | `float`, `f32`, `f64`, `number` | coercion | the value is representable | `E0209` |
| an integer literal | `i8` … `i64`, `u8` … `u64` | coercion | the value is in the range of that type | `E0209` |
| a float literal | `float` | default | always | — |
| a float literal | `f32`, `f64` | coercion | the value is representable in that format | `E0209` |
| a float literal | `int`, any integer type | — | never | `E0201` |
| a string literal | `string` | default | always | — |
| a string literal | a union of string-literal types | subtyping | the literal is one of the members | `E0201` |
| `T` | `T?` | subtyping | always; `null <: T?` likewise | — |
| `int`, `float` | `number` | subtyping | union membership, `number = int \| float` | — |
| a value with a string form | `string`, inside `${…}` | coercion | the operand is `int`, `float`, `boolean`, `string`, or `ToString` | `E0212` |

- The numeric rules concern **literals**, not values. A variable of type `int`
  has no adaptation to any other numeric type, however small its value is
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- A float literal never adapts to an integer type: `let i: int = 3.0` is an
  error. Narrowing a `float` is written with an explicit operation such as
  `Math.floor` or `Math.trunc`.
- A literal whose value does not fit its default type — an integer literal
  beyond `int` — is `E0209` in any context.
- `int` and `float` reach `number` by **union membership, not by coercion**:
  `number` is defined as `int | float`, so the subtype rule alone admits each
  member and nothing converts. The converse does not hold, because a `number`
  may be the other member of the union.
- `T <: T?` is likewise not a conversion: the value is unchanged, only its
  type widens. Narrowing an optional back to `T` is never implicit
  ([`../types/type-relations.md`](../types/type-relations.md)).

```xulo
let n: int = 42          // the literal is `int`
let d: number = 42       // the literal takes `number`
let w: u32 = 800         // the literal takes `u32`
let f: float = 42        // the literal takes `float`
let b: u8 = 300          // error[E0209]: 300 does not fit in `u8`
let i: int = 3.14        // error[E0201]: a float literal is not an `int`
let m: number = n        // fine: `int <: number`, no conversion
let s: Status = "active" // Status = "active" | "inactive"
```

## Numeric promotion vs coercion

Promotion is a rule about **operators**, not about assignment. In `+`, `-`,
`*`, `/`, `%`, `**`, and in the relational operators, operands of the
user-facing numeric layer combine as follows; the table is exhaustive, and a
combination it does not give is a compile-time error
([`../types/primitive-types.md`](../types/primitive-types.md)):

| Operands | Result |
|----------|--------|
| `int`, `int` | `int` |
| `float`, `float` | `float` |
| `int`, `float` (either order) | `float` |
| `number`, `int` (either order) | `number` |
| `number`, `float` (either order) | `float` |
| `number`, `number` | `number` |
| one fixed-bit type, the same fixed-bit type | that type |
| distinct fixed-bit types, or a fixed-bit type with any other numeric type | error `E0201` |

Constant division or remainder by zero, and constant results that overflow the
type, are compile-time errors (`E0210`); the same conditions with non-constant
operands are runtime traps.

Promotion stops at the operator. The result of `n + 0.5` is `float` and
assigning that result to a `float` binding needs nothing further — but the
promotion never carries a value across a checking site on its own:

```xulo
let sum = n + 0.5              // `int + float` has type `float`
let half = f / 2               // `float / int` has type `float`
let bad: float = n             // error[E0201]: `int` is not assignable to `float`
let ok: float = n * 1.0
```

`int` → `float` promotion happens **only inside arithmetic operators, never in
assignment**: `let f: float = someInt` is an error however the language would
combine the same two operands in an expression.

## String interpolation

Two positions convert a value to text, and they accept exactly the same
operands: the intrinsic `str(value)` and the interpolation `${value}` of a
template literal. The operand MUST be `int`, `float`, `boolean`, `string`, or a
type implementing the built-in `ToString` protocol; anything else is `E0212`
([`../expressions/literals.md`](../expressions/literals.md),
[`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).

```xulo
let label = `count=${count}`    // int renders as a numeral
let line = "total: " + str(n)   // `+` never converts; `str` does
let bad = `user: ${user}`       // error[E0212]: User has no string form
```

This is the only implicit conversion admitted for a type outside the primitive
numeric and string families: an `enum` or a `struct` carrying
`impl ToString` renders in `${…}` without an explicit call, and no other
complex type is converted implicitly. A template literal always has type
`string`, and nowhere else does the language convert to text — `+`
concatenates two `string`s and never converts its operands.

## What does NOT coerce

Every other conversion is rejected. If a row is not in the table of this
chapter, the conversion does not exist:

| Value type | Wanted type | Result | Written instead |
|------------|-------------|--------|-----------------|
| `int` (not a literal) | `float` | `E0201` | `n * 1.0`, or an operation that promotes |
| `int` (not a literal) | any fixed-bit integer | `E0201` | an operation on the fixed-bit type itself |
| `float` | `int`, any fixed-bit type | `E0201` | `Math.floor`, `Math.trunc`, `Math.round` |
| `number` | `int`, `float` | `E0201` | an explicit check, or `num * 1.0` for `float` |
| one fixed-bit type | another fixed-bit type | `E0201` | arithmetic within the target type |
| `string` | `int`, `float`, `boolean` | `E0201` | nothing converts; produce the value directly |
| `int`, `float`, `number` | `boolean` | `E0201` | a comparison: `n != 0` |
| `boolean` | anything | `E0201` | a `match` or `?:` that produces the target |
| `null` | `T` (not `T?`) | `E0201` | narrow first: `x ?? fallback` |
| `T?` | `T` | `E0201` | narrow first: `x ?? fallback` |
| `list<int>` | `list<float>` | `E0201` | map the elements explicitly |
| `(int, int)` | `(number, number)` | `E0201` | write a tuple literal, or convert element-wise |
| `map<K, V>` | `list<V>` | `E0201` | iterate and collect |
| `View?` | `View` | `E0201`; `E0702` in a component block | narrow first: `v ?? fallback` |

```xulo
let s: string = n        // error[E0201]: `int` is not assignable to `string`
let t: int = "42"        // error[E0201]: `string` is not assignable to `int`
let u: boolean = n       // error[E0201]: `int` is not assignable to `boolean`
let v: int = f           // error[E0201]: `float` is not assignable to `int`
let big: i64 = small     // error[E0201]: `i32` is not assignable to `i64`
let ys: list<float> = xs // error[E0201]: `list<int>` is not assignable to `list<float>`
let pr: (number, number) = pair   // error[E0201]: `(int, int)` vs `(number, number)`
```

Subtyping needs no listing and converts nothing: `T <: T?`, `null <: T?`,
union injection, and the covariance rules all happen without a conversion
([`../types/type-relations.md`](../types/type-relations.md)).

## Equality and comparison

`==` and `!=` perform no conversion of their operands. Both operands MUST have
a common type — equality requires one and creates none — and comparing two
types that have none is `E0211`
([`../expressions/operators.md`](../expressions/operators.md)):

```xulo
let same = a == b               // `a` and `b` share a type
let bit = mask == 0xff          // both operands are `int`
let wrong = n == "1"            // error[E0211]: no common type
```

- `null == null` is `true`; comparing `null` with a value requires that
  value's type to be optional or `unknown`, which is subtyping, not a
  conversion.
- An `unknown` operand makes `unknown` the common type of any pair it joins,
  so `x == y` and `x != null` are well-formed for `x : unknown` and any `y`.
  The result is `boolean`, and the comparison narrows nothing: `x` keeps type
  `unknown` in both branches of a following `if` — only a `match` type pattern
  narrows it ([`../expressions/control-flow.md`](../expressions/control-flow.md)).
- Strings compare lexicographically by Unicode code point, locale
  independently. Structural equality for `list`, `map`, `set`, structs,
enums, and tuples compares the contained values, each pair under
the same common-type rule.
- Relational operators follow the same operand rule as arithmetic promotion
  and convert nothing else: two operands are numeric with the promotion table
  applied, or both `string`.

## Acceptance

A conforming checker applies exactly the conversions in this chapter and no
others. Where this chapter and any example disagree, the tables win; an
example is non-normative ([`../README.md`](../README.md)).
