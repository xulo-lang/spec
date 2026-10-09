# Primitive Types

Primitive types are the built-in, non-composite types of Xulo: the base types `Boolean` and `String`, the two-layer numeric types, the `Null` and `Unit` types, and the built-in marker and range types `View` and `Range<T>`. They are defined by the language itself; a program may name them, annotate with them, and combine them, but no program can declare a new primitive type.

## Boolean

`Boolean` has exactly two values, written with the keywords `true` and `false`.

```xulo
let yes = true
let no = false
let ok = yes and !no
```

- The literals `true` and `false` have type `Boolean`.
- There is **no implicit conversion** between `Boolean` and any other type. In particular, numbers are not booleans: `0` and `1` are values of `Int`, and `if 0 { … }` is a compile-time error. There is no truthiness of any kind.
- The operands of `and`, `or`, and unary `!` MUST have type `Boolean`; otherwise a compile-time error is reported (see [`../expressions/operators.md`](../expressions/operators.md)). The same requirement applies to the condition of `if`, `while`, and the `?:` ternary.
- `==` and `!=` are defined for two `Boolean` operands. Arithmetic, ordering, and bitwise operators are not defined on `Boolean`.

## String

A `String` is an immutable sequence of Unicode scalar values, encoded as UTF-8. The characters of a `String` can never be changed in place: every operation that appears to produce a different string produces a new value.

```xulo
let a = "hello"
let b = 'world'
let c = a + " " + b
let greeting = `Hello, ${a}!`
```

- String literals are written with `"` or `'`. Neither form interpolates; the backtick template form interpolates `${expr}`. Literal and template syntax is specified in [`../expressions/literals.md`](../expressions/literals.md).
- The `+` operator concatenates two strings: both operands MUST have type `String`, and the result is `String`. Mixed operands are a compile-time error — `"count=" + 1` MUST NOT be written; render the value with a template — `` `count=${n}` `` — or a conversion function such as `str` (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- **Strings are not indexable.** `s[i]` is NOT a defined operation for a value of type `String` and is a compile-time error. The language defines no indexing, slicing, or character access on `String`; operations that inspect or derive strings are library-level and out of scope for this specification (see [`../README.md`](../README.md)).
- `==` and `!=` are defined for two `String` operands and compare by content. The quote form is irrelevant to both type and equality: `"hello" == 'hello'` is `true`.

## Numeric Types

Xulo has a **two-layer** numeric type system: a user-facing layer for everyday code and a formal fixed-bit layer for exact layout, FFI, and systems work. Both layers are ordinary types; nothing converts between them implicitly except the rules given below.

### User-facing layer

| Type | Description | Default for |
|------|-------------|-------------|
| `Number` | Generic numeric type; equivalent to the union `Int \| Float` | no literal form |
| `Int` | 64-bit signed integer | integer literals (`42`) |
| `Float` | 64-bit IEEE-754 binary64 floating point | float literals (`3.14`) |

`Number` is transparent: a value of type `Int` or `Float` is assignable to `Number`, and a `Number` is usable anywhere either is required. The converse does not hold — a `Number` is not assignable to `Int` or to `Float` without an explicit check, because it may be the other member of the union.

### Formal layer (fixed-bit types)

| Type | Width | Range |
|------|-------|-------|
| `I8` | 8-bit signed | -128 to 127 |
| `I16` | 16-bit signed | -32,768 to 32,767 |
| `I32` | 32-bit signed | -2^31 to 2^31 - 1 |
| `I64` | 64-bit signed | -2^63 to 2^63 - 1 |
| `U8` | 8-bit unsigned | 0 to 255 |
| `U16` | 16-bit unsigned | 0 to 65,535 |
| `U32` | 32-bit unsigned | 0 to 2^32 - 1 |
| `U64` | 64-bit unsigned | 0 to 2^64 - 1 |
| `F32` | 32-bit IEEE 754 binary32 | about 7 decimal digits of precision |
| `F64` | 64-bit IEEE 754 binary64 | about 15 decimal digits of precision |

Fixed-bit types are the types of choice for struct fields that describe memory, pixels, or a foreign interface:

```xulo
struct ImageDimensions {
  width: U32
  height: U32
  opacity: F32
}
```

Fixed-bit types map directly to C, Rust, and WASM ABI types:

| Xulo | C / Rust | WASM |
|------|----------|------|
| `I32` | `int32_t` | `I32` |
| `U32` | `uint32_t` | `I32` (unsigned) |
| `I64` | `int64_t` | `I64` |
| `U64` | `uint64_t` | `I64` (unsigned) |
| `F32` | `Float` | `F32` |
| `F64` | `double` | `F64` |

### Literal defaulting and coercion

| Literal form | Inferred type |
|--------------|---------------|
| `42` (decimal integer) | `Int` |
| `3.14` (float) | `Float` |
| `0xff` (hexadecimal integer) | `Int` |
| `0b1010` (binary integer) | `Int` |
| `0o77` (octal integer) | `Int` |

- An integer literal defaults to `Int` and a float literal to `Float`, even when no context requires it: `let x = 42` infers `Int`.
- When an integer or float literal is used where a fixed-bit type is required — a binding annotation, a call argument, a return value — and its value is within the range of that type, the literal is coerced to the fixed-bit type: `let w: U32 = 800` has type `U32`, and `let o: F32 = 0.8` has type `F32`.
- Coercion of non-literal values, and all other numeric conversions, are specified in [`../type-system/coercion.md`](../type-system/coercion.md). Outside of literal coercion and the arithmetic promotion rules below, numeric types do not convert implicitly: a value of type `Int` is not assignable to `Float`, and no value of type `Int` or `Float` is assignable to a fixed-bit type.
- **Overflow.** An integer operation (`Int` or any fixed-bit integer type) that exceeds the range of its result type is a compile-time error when every operand of the operation is a constant expression, and a runtime trap otherwise. A runtime trap is a panic: the program stops where the condition occurs, and there is no handler (see [`../memory-and-runtime.md`](../memory-and-runtime.md)). A literal that does not fit a fixed-bit type it is coerced into is likewise a compile-time error.
- Operations on `Float`, `F32`, and `F64` follow IEEE-754 semantics for their format.

### Arithmetic result types

All arithmetic operators (`+`, `-`, `*`, `/`, `%`, `**`) are typed by their operands:

| Left operand | Right operand | Result |
|--------------|---------------|--------|
| `Int` | `Int` | `Int` |
| `Float` | `Float` | `Float` |
| `Int` | `Float` | `Float` (promotion) |
| `Float` | `Int` | `Float` (promotion) |
| `Number` | `Number` | `Number` |
| `Number` | `Float` | `Float` |
| `Float` | `Number` | `Float` |
| `Number` | `Int` | `Number` |
| `Int` | `Number` | `Number` |
| fixed-bit `T` | `T`, or an in-range literal of `T`'s default type | `T` |

- Fixed-bit arithmetic stays in its own type: `U32 + U32` is `U32`, `F32 * F32` is `F32`. Mixing a fixed-bit type with any different numeric type — `U8 + U16`, `Int + U32`, `I32 + Float`, `Number + U32` — is a compile-time error; there is no implicit conversion between fixed-bit types or between a fixed-bit type and the user-facing layer (except the literal coercion described above). In the user-facing layer the table is exhaustive: a numeric combination it does not give is a compile-time error.
- Comparison operators (`<`, `>`, `<=`, `>=`) follow the same operand rules and produce `Boolean`.

## Null

`null` is both a literal — the keyword `null` — and a type: the type `Null`, whose only value is `null`.

```xulo
let nothing: String? = null
let present: String? = "xulo"
let name = present ?? "stranger"
```

- `null` is a member of every optional type: for every type `T`, `null` is assignable to `T?`, and `T?` is exactly `T | Null` (see [`composite-types.md`](composite-types.md)).
- Assigning `null` to a type that is not optional — `String`, `Int`, `List<Int>` — is a compile-time error. There is no implicit nullability.
- Null checking is strict. `null` is not a `Boolean` and is not interchangeable with `false`; a value of type `T?` MUST be tested against `null` or propagated with `??` and `?.` before it is used where `T` is required (see [`composite-types.md`](composite-types.md)).

## Unit

`Unit` is the type of expressions that produce no meaningful value. It has exactly one value, and the language provides no literal for that value — `Unit` is never written `()` (see [`composite-types.md`](composite-types.md)). Values of the type are produced by expressions such as a call to a function that declares no return type or a `return` with no value (see [`../statements/return-and-block.md`](../statements/return-and-block.md)).

```xulo
fn log(message: String) {
  print(message)
}
```

- A function with no declared return type returns `Unit`; declaring `: Unit` explicitly is equivalent to omitting the return type.
- An `async` function with no declared return type has evaluated type `fn(...): Task<Unit>`, and the built-in task type `Task<T>` is specified in [`function-types.md`](function-types.md).
- A `Unit`-valued expression MUST NOT be used where a value is required: as a binding initializer, as a call argument, as an operand, or as a condition. Its permitted uses are as an expression statement and as the result of a function that returns `Unit` (see [`../statements/expression-statements.md`](../statements/expression-statements.md)).

## Unknown

`Unknown` is the **top type**: every type is a subtype of it, so a value of
any type fits a position declared `Unknown`. It is how a deliberately
heterogeneous slot is written — `List<Unknown>` holds values of any type, and
`Map<String, Unknown>` maps strings to anything.

- Subtyping runs one way only: `T <: Unknown` for every `T`, and `Unknown <: T`
  only when `T` is `Unknown`. Nothing else relates `Unknown` to another type
  ([`type-relations.md`](type-relations.md)).
- An `Unknown` value supports no operation except `==`/`!=`, the type test
  `is`, and `match`: member access on it is `E0102`, and using it as the
  operand of any other operator is `E0201`. The type records that a value
  exists, not what it is.
- `==` and `!=` accept an `Unknown` operand, because `Unknown` is the common
  type of any pair it joins, so `x == null` tests for absence. The test yields
  a `Boolean` like any other comparison and does **not** narrow `x`: a value of
  type `Unknown` stays `Unknown` in both branches, and neither does `x != null`.
- Inference never produces `Unknown` on its own: a literal is `Int`,
  `Float`, `String`, `Boolean`, or `null`, or a type the context adapts it to.
  `Unknown` appears only where it is written — an annotation, a type argument,
  a field type — or as the tested type of a `match` type pattern or of an `is`
  test.
- The ways to *use* such a value are `match` with a type pattern, which
  narrows it for that arm (`String s => …`), and `is`, which narrows the
  branches of an `if` (`data is String`)
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)).

## View

`View` is the built-in marker type produced by components: a component is a function whose declared return type is `View` (see [`../components/README.md`](../components/README.md)).

```xulo
fn Greeting(name: String): View {
  Text(`Hello, ${name}`)
}
```

Values of type `View` are opaque to the core type system beyond being renderable: apart from appearing as a component's result, as an element of a rendered `List<View>`, and as a child of another component, the language defines no field access, method call, indexing, or conversion on `View`. The concrete component vocabulary is provided by the UI library, not by this specification.

## Range<T>

`Range<T>` is the built-in generic type produced by the range operators `..<` (half-open) and `...` (closed):

```xulo
let a = 0..<10     // Range<Int>: excludes 10
let b = 1...5      // Range<Int>: includes 5
for i in a {
  print(i)
}
```

- The operands of a range operator MUST have the same numeric type `T`; the result type is `Range<T>`.
- A `Range<T>` is iterable: `for x in range` visits each value of type `T` from the lower bound up to — but not including, for `..<`; including, for `...` — the upper bound.
- Ranges are values; a `Range<T>` may be bound, passed, and returned like any other value. Range patterns in `match` are specified in [`../expressions/README.md`](../expressions/README.md).

## Reserved Type Names

The names of the built-in types are reserved for the language. A module-scope `struct`, `enum`, `trait`, or `type` declaration MUST NOT use any of the following names:

`Boolean`, `String`, `Int`, `Float`, `Number`, `I8`, `I16`, `I32`, `I64`, `U8`, `U16`, `U32`, `U64`, `F32`, `F64`, `Null`, `Unit`, `Unknown`, `List`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error`

Declaring a type with one of these names at module scope is a compile-time error. `List`, `Map`, and `Set` are specified in [`composite-types.md`](composite-types.md), `Task<T>` in [`function-types.md`](function-types.md), `Result` and `Error` in [`../error-handling.md`](../error-handling.md), and `View` in [`../components/view-syntax.md`](../components/view-syntax.md); every other name above is specified in this file. Elsewhere — as a type parameter name or an ordinary value binding — a built-in type name is an ordinary identifier, though using one conflicts with the naming conventions in [`../names.md`](../names.md).
