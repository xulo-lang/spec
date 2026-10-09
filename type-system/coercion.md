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
  takes its default type — `Int`, `Float`, `String`, `Boolean` — and the
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
| an integer literal | `Int` | default | always | — |
| an integer literal | `Float`, `F32`, `F64`, `Number` | coercion | the value is representable | `E0209` |
| an integer literal | `I8` … `I64`, `U8` … `U64` | coercion | the value is in the range of that type | `E0209` |
| a float literal | `Float` | default | always | — |
| a float literal | `F32`, `F64` | coercion | the value is representable in that format | `E0209` |
| a float literal | `Int`, any integer type | — | never | `E0201` |
| a string literal | `String` | default | always | — |
| a string literal | a union of string-literal types | subtyping | the literal is one of the members | `E0201` |
| `T` | `T?` | subtyping | always; `Null <: T?` likewise | — |
| `Int`, `Float` | `Number` | subtyping | union membership, `Number = Int \| Float` | — |
| a value with a string form | `String`, inside `${…}` | coercion | the operand is `Int`, `Float`, `Boolean`, `String`, or `ToString` | `E0212` |

- The numeric rules concern **literals**, not values. A variable of type `Int`
  has no adaptation to any other numeric type, however small its value is
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- A float literal never adapts to an integer type: `let i: Int = 3.0` is an
  error. Narrowing a `Float` is written with an explicit operation such as
  `Math.floor` or `Math.trunc`.
- A literal whose value does not fit its default type — an integer literal
  beyond `Int` — is `E0209` in any context.
- `Int` and `Float` reach `Number` by **union membership, not by coercion**:
  `Number` is defined as `Int | Float`, so the subtype rule alone admits each
  member and nothing converts. The converse does not hold, because a `Number`
  may be the other member of the union.
- `T <: T?` is likewise not a conversion: the value is unchanged, only its
  type widens. Narrowing an optional back to `T` is never implicit
  ([`../types/type-relations.md`](../types/type-relations.md)).

```xulo
let n: Int = 42          // the literal is `Int`
let d: Number = 42       // the literal takes `Number`
let w: U32 = 800         // the literal takes `U32`
let f: Float = 42        // the literal takes `Float`
let b: U8 = 300          // error[E0209]: 300 does not fit in `U8`
let i: Int = 3.14        // error[E0201]: a float literal is not an `Int`
let m: Number = n        // fine: `Int <: Number`, no conversion
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
| `Int`, `Int` | `Int` |
| `Float`, `Float` | `Float` |
| `Int`, `Float` (either order) | `Float` |
| `Number`, `Int` (either order) | `Number` |
| `Number`, `Float` (either order) | `Float` |
| `Number`, `Number` | `Number` |
| one fixed-bit type, the same fixed-bit type | that type |
| distinct fixed-bit types, or a fixed-bit type with any other numeric type | error `E0201` |

Constant division or remainder by zero, and constant results that overflow the
type, are compile-time errors (`E0210`); the same conditions with non-constant
operands are runtime traps.

Promotion stops at the operator. The result of `n + 0.5` is `Float` and
assigning that result to a `Float` binding needs nothing further — but the
promotion never carries a value across a checking site on its own:

```xulo
let sum = n + 0.5              // `Int + Float` has type `Float`
let half = f / 2               // `Float / Int` has type `Float`
let bad: Float = n             // error[E0201]: `Int` is not assignable to `Float`
let ok: Float = n * 1.0
```

`Int` → `Float` promotion happens **only inside arithmetic operators, never in
assignment**: `let f: Float = someInt` is an error however the language would
combine the same two operands in an expression.

## String interpolation

Two positions convert a value to text, and they accept exactly the same
operands: the intrinsic `str(value)` and the interpolation `${value}` of a
template literal. The operand MUST be `Int`, `Float`, `Boolean`, `String`, or a
type implementing the built-in `ToString` protocol; anything else is `E0212`
([`../expressions/literals.md`](../expressions/literals.md),
[`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).

```xulo
let label = `count=${count}`    // Int renders as a numeral
let line = "total: " + str(n)   // `+` never converts; `str` does
let bad = `user: ${user}`       // error[E0212]: User has no string form
```

This is the only implicit conversion admitted for a type outside the primitive
numeric and string families: an `enum` or a `struct` carrying
`impl ToString` renders in `${…}` without an explicit call, and no other
complex type is converted implicitly. A template literal always has type
`String`, and nowhere else does the language convert to text — `+`
concatenates two `String`s and never converts its operands.

## What does NOT coerce

Every other conversion is rejected. If a row is not in the table of this
chapter, the conversion does not exist:

| Value type | Wanted type | Result | Written instead |
|------------|-------------|--------|-----------------|
| `Int` (not a literal) | `Float` | `E0201` | `n * 1.0`, or an operation that promotes |
| `Int` (not a literal) | any fixed-bit integer | `E0201` | an operation on the fixed-bit type itself |
| `Float` | `Int`, any fixed-bit type | `E0201` | `Math.floor`, `Math.trunc`, `Math.round` |
| `Number` | `Int`, `Float` | `E0201` | an explicit check, or `num * 1.0` for `Float` |
| one fixed-bit type | another fixed-bit type | `E0201` | arithmetic within the target type |
| `String` | `Int`, `Float`, `Boolean` | `E0201` | nothing converts; produce the value directly |
| `Int`, `Float`, `Number` | `Boolean` | `E0201` | a comparison: `n != 0` |
| `Boolean` | anything | `E0201` | a `match` or `?:` that produces the target |
| `Null` | `T` (not `T?`) | `E0201` | narrow first: `x ?? fallback` |
| `T?` | `T` | `E0201` | narrow first: `x ?? fallback` |
| `List<Int>` | `List<Float>` | `E0201` | map the elements explicitly |
| `(Int, Int)` | `(Number, Number)` | `E0201` | write a tuple literal, or convert element-wise |
| `Map<K, V>` | `List<V>` | `E0201` | iterate and collect |
| `View?` | `View` | `E0201`; `E0702` in a component block | narrow first: `v ?? fallback` |

```xulo
let s: String = n        // error[E0201]: `Int` is not assignable to `String`
let t: Int = "42"        // error[E0201]: `String` is not assignable to `Int`
let u: Boolean = n       // error[E0201]: `Int` is not assignable to `Boolean`
let v: Int = f           // error[E0201]: `Float` is not assignable to `Int`
let big: I64 = small     // error[E0201]: `I32` is not assignable to `I64`
let ys: List<Float> = xs // error[E0201]: `List<Int>` is not assignable to `List<Float>`
let pr: (Number, Number) = pair   // error[E0201]: `(Int, Int)` vs `(Number, Number)`
```

Subtyping needs no listing and converts nothing: `T <: T?`, `Null <: T?`,
union injection, and the covariance rules all happen without a conversion
([`../types/type-relations.md`](../types/type-relations.md)).

## Equality and comparison

`==` and `!=` perform no conversion of their operands. Both operands MUST have
a common type — equality requires one and creates none — and comparing two
types that have none is `E0211`
([`../expressions/operators.md`](../expressions/operators.md)):

```xulo
let same = a == b               // `a` and `b` share a type
let bit = mask == 0xff          // both operands are `Int`
let wrong = n == "1"            // error[E0211]: no common type
```

- `null == null` is `true`; comparing `null` with a value requires that
  value's type to be optional or `Unknown`, which is subtyping, not a
  conversion.
- An `Unknown` operand makes `Unknown` the common type of any pair it joins,
  so `x == y` and `x != null` are well-formed for `x : Unknown` and any `y`.
  The result is `Boolean`, and the comparison narrows nothing: `x` keeps type
  `Unknown` in both branches of a following `if` — only a `match` type pattern
  narrows it ([`../expressions/control-flow.md`](../expressions/control-flow.md)).
- Strings compare lexicographically by Unicode code point, locale
  independently. Structural equality for `List`, `Map`, `Set`, structs,
enums, and tuples compares the contained values, each pair under
the same common-type rule.
- Relational operators follow the same operand rule as arithmetic promotion
  and convert nothing else: two operands are numeric with the promotion table
  applied, or both `String`.

## Acceptance

A conforming checker applies exactly the conversions in this chapter and no
others. Where this chapter and any example disagree, the tables win; an
example is non-normative ([`../README.md`](../README.md)).
