# Intrinsic Functions — Delta: String Form — Display, Debug, and Format

> **Delta** — edits the first `print` bullet; inserts the consult
> paragraph, its outcome table, and the type-timing rule between the
> bullets and the table; rewrites the table's introduction (which defines
> `structural positions`) and adds the completion rows and a struct example
> to the Output block; and replaces the whole "Conversion" section and the
> `str` row of the signature index of `builtins/intrinsic-functions.md`.

## Output

The first bullet of the section becomes:

- The signature accepts **any type**: `print` and `println` are the only
  facilities that render a structured value directly. The signature's
  openness is not a promise about every operand — the paragraph below makes
  a `struct` with neither implementation, or a value typed `unknown`, a
  compile-time error (`E0212`).

A paragraph, an outcome table, and a timing rule are inserted between the
section's bullets and the table's introduction:

A value of a user-declared type — a `struct` or an `enum` — consults an
implementation before any table, wherever the rendering reaches it: as the
printed value itself or in any **structural position** of the type being
rendered — a container's elements, an optional's value, a tuple's
elements, a `Result`'s slots, an enum's payloads. The outcome in full:

| The value's type | implements `Display` | implements `Debug` | neither |
|------------------|----------------------|--------------------|---------|
| a user `struct` | its `Display` at the default `Format` | its `Debug` | a compile-time error (`E0212`) |
| a value typed `unknown` — as the operand or in any structural position | not consulted | not consulted | rejected at the call, before any rendering: `E0212` — the same error interpolating it gives |
| a user `enum` | its `Display` at the default `Format` | its `Debug` | the enum row of the table below |
| any other type | not consulted | not consulted | the type's row of the table below |

A type-parameter operand is decided at the call that supplies the type
argument — the site that checks bounds (`E0801`) — so a generic body
admits `print(x)` for an unbounded `T`, and the error, if any, appears
with the type that is actually supplied. A type known at the call —
`unknown` included, by the row above — is decided there, at once.

The table's own introduction then becomes:

Every other rendering is computed recursively by this table; the rendering
of a structured value is built from the renderings of its **structural
positions** — a container's elements, an optional's `T`, a tuple's
elements, a `Result`'s `T` and `E` slots, and an enum's payloads — which
is exactly where the consult above applies.

The rendering table itself gains rows, placed with their kinds; with the
consult paragraph they cover every type a value or a derived field may
have — `unknown` alone is rejected rather than rendered (above):

| Value | Rendering |
|-------|-----------|
| `number` | as its runtime member — the `int` row or the `float` row |
| `i8` … `i64`, `u8` … `u64`, `usize`, `isize` | the decimal numeral of the value — exactly the `int` row's spelling |
| `f32`, `f64` | `f64` as the `float` row; `f32` as the shortest positional decimal numeral that reads back as the same `f32`, the special values rendering as `NaN`, `Infinity`, and `-Infinity` |
| `unit` | `unit` |
| an optional `T?` | `null` as the `null` row for a null value, otherwise by the rules that apply to its `T` |
| `Range<T>` | `start..<end` or `start...end`, per the value's own form, bounds by this table (`0..<10`, `1...5`) |
| `Result<T, E>` | `Ok(…)` or `Err(…)`, the payload by this table |
| a tuple `(T₁, …)` | `(`, the element renderings separated by `, `, `)` — `(1, x)` |
| `Error` | `Error(…)`, the message by this table — `Error(timed out)` |
| `Format` | `Format(precision: …, alternate: …)`, the fields by this table |
| `Ptr` | `<ptr>` |

The example block of the section gains a struct example, appended to its
lines:

```xulo
#Derive(Debug)
struct Point { x: int, y: int }
print(Point(x: 1, y: 2))         // Point(x: 1, y: 2)
```

The whole "Conversion" section becomes:

## Conversion

| Signature | Result |
|-----------|--------|
| `str(value: int \| float \| bool \| string \| Display): string` | the string form of `value` |

- The operand MUST be a base type — `int`, `float`, `bool`, `string` — or a
  value whose type implements the built-in `Display` protocol
  ([`prelude.md`](prelude.md#built-in-protocols)). In the signature above,
  `Display` denotes the type of the values that implement the protocol.
- `str(value)` and the interpolation `${value}` accept exactly the same
  operands: where one is well-formed so is the other
  ([`../expressions/literals.md`](../expressions/literals.md)).
- A base type is rendered exactly as in the table under [Output](#output); for
  a `Display` implementor the result is the value of `toString` on the
  operand — its `display` at the default `Format`.
- Any other operand is a compile-time error (`E0212`) — a `List`, an `Array`, a
  `Map`, a `Set`, a `struct` or `enum` with no `impl Display`, or `null`
  ([`../type-system/errors.md`](../type-system/errors.md)).

There is no implicit conversion anywhere else in the language: `+` concatenates
two strings and never converts its operands, so a mixed expression is written
with `str` ([`../expressions/operators.md`](../expressions/operators.md)).

```xulo
let name = "ada"
let n = 42
let label = "count=" + str(n)          // "count=42"
let who = `hello ${name}`              // interpolation needs no str
let maybe: int? = null
let absent = str(maybe ?? 0)           // "0": narrow an optional first
let bad = str([1, 2])                  // error: List<int> has no string form
```

In the chapter's complete signature index, the `str` row becomes:

| `str` | `str(value: int \| float \| bool \| string \| Display): string` | [Conversion](#conversion) |
