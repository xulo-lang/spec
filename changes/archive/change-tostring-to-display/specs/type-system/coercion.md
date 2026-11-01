# Coercion and Implicit Conversion — Delta: String Form — Display, Debug, and Format

> **Delta** — renames the protocol in the last row of the coercion table and
> replaces the "string interpolation" section of
> `type-system/coercion.md`. The conversion admitted is unchanged; only the
> protocol's name moves.

The final row of the coercion table becomes:

| a value with a string form | `string`, inside `${…}` | coercion | the operand is `int`, `float`, `bool`, `string`, or `Display` | `E0212` |

## string interpolation

Two positions convert a value to text, and they accept exactly the same
operands: the intrinsic `str(value)` and the interpolation `${value}` of a
template literal. The operand MUST be `int`, `float`, `bool`, `string`, or a
type implementing the built-in `Display` protocol; anything else is `E0212`
([`../expressions/literals.md`](../expressions/literals.md),
[`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).

```xulo
let label = `count=${count}`    // int renders as a numeral
let line = "total: " + str(n)   // `+` never converts; `str` does
let bad = `user: ${user}`       // error[E0212]: User has no string form
```

This is the only implicit conversion admitted for a type outside the primitive
numeric and string families: an `enum` or a `struct` carrying
`impl Display` renders in `${…}` without an explicit call, and no other
complex type is converted implicitly. A template literal always has type
`string`, and nowhere else does the language convert to text — `+`
concatenates two `string`s and never converts its operands.
