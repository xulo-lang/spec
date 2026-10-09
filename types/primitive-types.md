# Primitive Types

Primitive types are the built-in, non-composite types of Xulo: the base types `boolean` and `string`, the two-layer numeric types, the types of the `null` and `unit` values, and the built-in marker and range types `View` and `Range<T>`. They are defined by the language itself; a program may name them, annotate with them, and combine them, but no program can declare a new primitive type.

## boolean

`boolean` has exactly two values, written with the keywords `true` and `false`.

```xulo
let yes = true
let no = false
let ok = yes and !no
```

- The literals `true` and `false` have type `boolean`.
- There is **no implicit conversion** between `boolean` and any other type. In particular, numbers are not booleans: `0` and `1` are values of `int`, and `if 0 { … }` is a compile-time error. There is no truthiness of any kind.
- The operands of `and`, `or`, and unary `!` MUST have type `boolean`; otherwise a compile-time error is reported (see [`../expressions/operators.md`](../expressions/operators.md)). The same requirement applies to the condition of `if`, `while`, and the `?:` ternary.
- `==` and `!=` are defined for two `boolean` operands. Arithmetic, ordering, and bitwise operators are not defined on `boolean`.

## string

A `string` is an immutable sequence of Unicode scalar values, encoded as UTF-8. The characters of a `string` can never be changed in place: every operation that appears to produce a different string produces a new value.

```xulo
let a = "hello"
let b = 'world'
let c = a + " " + b
let greeting = `Hello, ${a}!`
```

- String literals are written with `"` or `'`. Neither form interpolates; the backtick template form interpolates `${expr}`. Literal and template syntax is specified in [`../expressions/literals.md`](../expressions/literals.md).
- The `+` operator concatenates two strings: both operands MUST have type `string`, and the result is `string`. Mixed operands are a compile-time error — `"count=" + 1` MUST NOT be written; render the value with a template — `` `count=${n}` `` — or a conversion function such as `str` (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- **Strings are not indexable.** `s[i]` is NOT a defined operation for a value of type `string` and is a compile-time error. The language defines no indexing, slicing, or character access on `string`; operations that inspect or derive strings are library-level and out of scope for this specification (see [`../README.md`](../README.md)).
- `==` and `!=` are defined for two `string` operands and compare by content.

## Numeric Types

Xulo has a **two-layer** numeric type system: a user-facing layer for everyday code and a formal fixed-bit layer for exact layout, FFI, and systems work. Both layers are ordinary types; nothing converts between them implicitly except the rules given below.

### User-facing layer

| Type | Description | Default for |
|------|-------------|-------------|
| `number` | Generic numeric type; equivalent to the union `int \| float` | no literal form |
| `int` | 64-bit signed integer | integer literals (`42`) |
| `float` | 64-bit IEEE-754 binary64 floating point | float literals (`3.14`) |

`number` is transparent: a value of type `int` or `float` is assignable to `number`, and a `number` is usable anywhere either is required. The converse does not hold — a `number` is not assignable to `int` or to `float` without an explicit check, because it may be the other member of the union.

### Formal layer (fixed-bit types)

| Type | Width | Range |
|------|-------|-------|
| `i8` | 8-bit signed | -128 to 127 |
| `i16` | 16-bit signed | -32,768 to 32,767 |
| `i32` | 32-bit signed | -2^31 to 2^31 - 1 |
| `i64` | 64-bit signed | -2^63 to 2^63 - 1 |
| `u8` | 8-bit unsigned | 0 to 255 |
| `u16` | 16-bit unsigned | 0 to 65,535 |
| `u32` | 32-bit unsigned | 0 to 2^32 - 1 |
| `u64` | 64-bit unsigned | 0 to 2^64 - 1 |
| `f32` | 32-bit IEEE 754 binary32 | about 7 decimal digits of precision |
| `f64` | 64-bit IEEE 754 binary64 | about 15 decimal digits of precision |

Fixed-bit types are the types of choice for struct fields that describe memory, pixels, or a foreign interface:

```xulo
struct ImageDimensions {
  width: u32
  height: u32
  opacity: f32
}
```

Fixed-bit types map directly to C, Rust, and WASM ABI types:

| Xulo | C / Rust | WASM |
|------|----------|------|
| `i32` | `int32_t` | `i32` |
| `u32` | `uint32_t` | `i32` (unsigned) |
| `i64` | `int64_t` | `i64` |
| `u64` | `uint64_t` | `i64` (unsigned) |
| `f32` | `float` | `f32` |
| `f64` | `double` | `f64` |

### Literal defaulting and coercion

| Literal form | Inferred type |
|--------------|---------------|
| `42` (decimal integer) | `int` |
| `3.14` (float) | `float` |
| `0xff` (hexadecimal integer) | `int` |
| `0b1010` (binary integer) | `int` |
| `0o77` (octal integer) | `int` |

- An integer literal defaults to `int` and a float literal to `float`, even when no context requires it: `let x = 42` infers `int`.
- When an integer or float literal is used where a fixed-bit type is required — a binding annotation, a call argument, a return value — and its value is within the range of that type, the literal is coerced to the fixed-bit type: `let w: u32 = 800` has type `u32`, and `let o: f32 = 0.8` has type `f32`.
- Coercion of non-literal values, and all other numeric conversions, are specified in [`../type-system/coercion.md`](../type-system/coercion.md). Outside of literal coercion and the arithmetic promotion rules below, numeric types do not convert implicitly: a value of type `int` is not assignable to `float`, and no value of type `int` or `float` is assignable to a fixed-bit type.
- **Overflow.** An integer operation (`int` or any fixed-bit integer type) that exceeds the range of its result type is a compile-time error when every operand of the operation is a constant expression, and a runtime trap otherwise. A literal that does not fit a fixed-bit type it is coerced into is likewise a compile-time error.
- Operations on `float`, `f32`, and `f64` follow IEEE-754 semantics for their format.

### Arithmetic result types

All arithmetic operators (`+`, `-`, `*`, `/`, `%`, `**`) are typed by their operands:

| Left operand | Right operand | Result |
|--------------|---------------|--------|
| `int` | `int` | `int` |
| `float` | `float` | `float` |
| `int` | `float` | `float` (promotion) |
| `float` | `int` | `float` (promotion) |
| `number` | `number` | `number` |
| `number` | `float` | `float` |
| `float` | `number` | `float` |
| `number` | `int` | `number` |
| `int` | `number` | `number` |
| fixed-bit `T` | `T`, or an in-range literal of `T`'s default type | `T` |

- Fixed-bit arithmetic stays in its own type: `u32 + u32` is `u32`, `f32 * f32` is `f32`. Mixing a fixed-bit type with any different numeric type — `u8 + u16`, `int + u32`, `i32 + float`, `number + u32` — is a compile-time error; there is no implicit conversion between fixed-bit types or between a fixed-bit type and the user-facing layer (except the literal coercion described above). In the user-facing layer the table is exhaustive: a numeric combination it does not give is a compile-time error.
- Comparison operators (`<`, `>`, `<=`, `>=`) follow the same operand rules and produce `boolean`.

## null

`null` is both a literal — the keyword `null` — and a type: the type `null`, whose only value is `null`.

```xulo
let nothing: string? = null
let present: string? = "xulo"
let name = present ?? "stranger"
```

- `null` is a member of every optional type: for every type `T`, `null` is assignable to `T?`, and `T?` is exactly `T | null` (see [`composite-types.md`](composite-types.md)).
- Assigning `null` to a type that is not optional — `string`, `int`, `list<int>` — is a compile-time error. There is no implicit nullability.
- Null checking is strict. `null` is not a `boolean` and is not interchangeable with `false`; a value of type `T?` MUST be tested against `null` or propagated with `??` and `?.` before it is used where `T` is required (see [`composite-types.md`](composite-types.md)).

## unit

`unit` is the type of expressions that produce no meaningful value. It has exactly one value, and the language provides no literal for that value: it is produced by expressions such as a call to a function that declares no return type.

```xulo
fn log(message: string) {
  print(message)
}
```

- A function with no declared return type returns `unit`; declaring `: unit` explicitly is equivalent to omitting the return type.
- An `async` function with no declared return type has evaluated type `fn(...): Task<unit>`, and the built-in task type `Task<T>` is specified in [`function-types.md`](function-types.md).
- A `unit`-valued expression MUST NOT be used where a value is required: as a binding initializer, as a call argument, as an operand, or as a condition. Its permitted uses are as an expression statement and as the result of a function that returns `unit` (see [`../statements/expression-statements.md`](../statements/expression-statements.md)).

## unknown

`unknown` is the **top type**: every type is a subtype of it, so a value of
any type fits a position declared `unknown`. It is how a deliberately
heterogeneous slot is written — `list<unknown>` holds values of any type, and
`map<string, unknown>` maps strings to anything.

- Subtyping runs one way only: `T <: unknown` for every `T`, and `unknown <: T`
  only when `T` is `unknown`. Nothing else relates `unknown` to another type
  ([`type-relations.md`](type-relations.md)).
- An `unknown` value supports no operation except `==`/`!=` and `match`:
  member access on it is `E0102`, and using it as the operand of any other
  operator is `E0201`. The type records that a value exists, not what it is.
- `==` and `!=` accept an `unknown` operand, because `unknown` is the common
  type of any pair it joins, so `x == null` tests for absence. The test yields
  a `boolean` like any other comparison and does **not** narrow `x`: a value of
  type `unknown` stays `unknown` in both branches, and neither does `x != null`.
- Inference never produces `unknown` on its own: a literal is `int`,
  `float`, `string`, `boolean`, or `null`, or a type the context adapts it to.
  `unknown` appears only where it is written — an annotation, a type argument,
  a field type — or as the tested type of a `match` type pattern.
- The one way to *use* such a value is `match` with a type pattern, which
  narrows it for that arm (`string s => …`)
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)).

## View

`View` is the built-in marker type produced by components: a component is a function whose declared return type is `View` (see [`../components/README.md`](../components/README.md)).

```xulo
fn Greeting(name: string): View {
  Text(`Hello, ${name}`)
}
```

Values of type `View` are opaque to the core type system beyond being renderable: apart from appearing as a component's result, as an element of a rendered `list<View>`, and as a child of another component, the language defines no field access, method call, indexing, or conversion on `View`. The concrete component vocabulary is provided by the UI library, not by this specification.

## Range<T>

`Range<T>` is the built-in generic type produced by the range operators `..<` (half-open) and `...` (closed):

```xulo
let a = 0..<10     // Range<int>: excludes 10
let b = 1...5      // Range<int>: includes 5
for i in a {
  print(i)
}
```

- The operands of a range operator MUST have the same numeric type `T`; the result type is `Range<T>`.
- A `Range<T>` is iterable: `for x in range` visits each value of type `T` from the lower bound up to — but not including, for `..<`; including, for `...` — the upper bound.
- Ranges are values; a `Range<T>` may be bound, passed, and returned like any other value. Range patterns in `match` are specified in [`../expressions/README.md`](../expressions/README.md).

## Reserved Type Names

The names of the built-in types are reserved for the language. A module-scope `struct`, `enum`, `trait`, or `type` declaration MUST NOT use any of the following names:

`boolean`, `string`, `int`, `float`, `number`, `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64`, `null`, `unit`, `unknown`, `list`, `map`, `set`, `Task`, `Range`, `Result`, `View`, `Error`

Declaring a type with one of these names at module scope is a compile-time error. Elsewhere — as a type parameter name or an ordinary value binding — a built-in type name is an ordinary identifier, though using one conflicts with the naming conventions in [`../names.md`](../names.md).
