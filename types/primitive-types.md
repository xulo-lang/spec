# Primitive Types

Primitive types are the built-in, non-composite types of Xulo. Two classes
spell differently. The **primitive value types** — the base types `bool`
and `string`, the numeric family `number`, `int`, `float`, and the
fixed-bit types `i8`…`u64`, `f32`, `f64`, and the singletons `null`,
`unit`, `unknown` — are reserved keywords: a program may use them in type
position and nowhere else, and no declaration may bear their spelling
([`../lexical-structure.md`](../lexical-structure.md)). The **marker and
generic primitives** — `View` and `Range<T>` — keep UpperCamelCase: `View`
is the return type of every component, `Range<T>` is generic, and both are
names a reader meets beside user types. No program can declare a new
primitive type of either class.

## bool

`bool` has exactly two values, written with the keywords `true` and `false`.

```xulo
let yes = true
let no = false
let ok = yes and !no
```

- The literals `true` and `false` have type `bool`.
- There is **no implicit conversion** between `bool` and any other type. In particular, numbers are not booleans: `0` and `1` are values of `int`, and `if 0 { … }` is a compile-time error (`E0201`). There is no truthiness of any kind.
- The operands of `and`, `or`, and unary `!` MUST have type `bool`; otherwise a compile-time error is reported (`E0201`; see [`../expressions/operators.md`](../expressions/operators.md)). The same requirement applies to the condition of `if`, `while`, and the `?:` ternary.
- `==` and `!=` are defined for two `bool` operands. Arithmetic, ordering, and bitwise operators are not defined on `bool`.

## string

A `string` is an immutable sequence of Unicode scalar values, encoded as UTF-8. The characters of a `string` can never be changed in place: every operation that appears to produce a different string produces a new value.

```xulo
let a = "hello"
let b = 'world'
let c = a + " " + b
let greeting = `Hello, ${a}!`
```

- A string literal is written with `"` or `'`. Neither form interpolates; the backtick template form interpolates `${expr}`. Literal and template syntax is specified in [`../expressions/literals.md`](../expressions/literals.md).
- The `+` operator concatenates two strings: both operands MUST have type `string`, and the result is `string`. Mixed operands are a compile-time error (`E0227`) — `"count=" + 1` MUST NOT be written; render the value with a template — `` `count=${n}` `` — or a conversion function such as `str` (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- **Strings are not indexable.** `s[i]` is NOT a defined operation for a value
  of type `string` and is a compile-time error (`E0215`). The language defines
  no indexing syntax on `string`; a string is inspected through its inherent
  members:
  - `.len` — a read-only property of type `int`, the number of Unicode
    scalar values;
  - `.byteLen` — a read-only property of type `int`, the number of UTF-8
    bytes, O(1) and allocation-free;
  - `.codeAt(i: int)` — the Unicode scalar value at index `i`, counting from
    0 in scalar order; type `int`;
  - `.slice(start: int, end: int)` — the substring containing the scalars at
    indices `start … end − 1`; type `string`;
  - `.bytes()` — a `List<u8>` holding the UTF-8 bytes, in order.
- Indices count Unicode scalar values, not bytes or UTF-16 code units;
  `codeAt` yields the scalar's value — its code point — as an `int`. An index
  outside `0 … s.len − 1`, slice bounds outside `0 … s.len`, or
  `start > end` is a runtime failure
  ([`../memory-and-runtime.md`](../memory-and-runtime.md), closed tables);
  `s.slice(n, n)` is the empty string for every `n` in range, and
  `s.slice(0, s.len)` is `s` itself.
- `==` and `!=` are defined for two `string` operands and compare by content. The quote form is irrelevant to both type and equality: `"hello" == 'hello'` is `true`.

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
| `usize` | `size_t` / `usize` | `i32` (32-bit) or `i64` (64-bit), the pointer width |
| `isize` | `ptrdiff_t` / `isize` | `i32` (32-bit) or `i64` (64-bit), the pointer width |

### Pointer-width integers

| Type | Width | Range |
|------|-------|-------|
| `usize` | `W` bits, unsigned — the platform's pointer width, implementation-defined | 0 to 2^W − 1 |
| `isize` | `W` bits, signed — the same `W` | −2^(W−1) to 2^(W−1) − 1 |

- `usize` and `isize` are formal-layer types whose width is the width of the
  platform's pointer: sizes, capacities, indices, and C's `size_t` and
  `ptrdiff_t`.
- The value of `W` is implementation-defined and an implementation MUST
  document it (32 or 64 in practice), together with the ABI document it comes
  from ([`../memory-and-runtime.md`](../memory-and-runtime.md)).
- A literal used where `usize` or `isize` is required follows the fixed-bit
  literal rule below: `let n: usize = 8` has type `usize`, and a literal out
  of range is a compile-time error (`E0209`).
- Arithmetic follows the fixed-bit promotion rule below: `usize + usize` is
  `usize`; mixing either type with any other numeric type — fixed-bit of
  another width, `int`, `number` — is a compile-time error (`E0201`), since
  a signed and an unsigned quantity of implementation-defined width have no
  defined mixed result. There is no coercion between `usize` and `isize`, or
  between either and any fixed-bit type ([`../type-system/coercion.md`](../type-system/coercion.md)).

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
- **Overflow.** An integer operation (`int` or any fixed-bit integer type) that exceeds the range of its result type is a compile-time error (`E0210`) when every operand of the operation is a constant expression, and a runtime trap otherwise. A runtime trap is a panic: the program stops where the condition occurs, and there is no handler (see [`../memory-and-runtime.md`](../memory-and-runtime.md)). A literal that does not fit a fixed-bit type it is coerced into is likewise a compile-time error (`E0209`). These rules govern the operators; the `Math::wrapping*` intrinsics define wrapping results and never trap (see [`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).
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

- Fixed-bit arithmetic stays in its own type: `u32 + u32` is `u32`, `f32 * f32` is `f32`. Mixing a fixed-bit type with any different numeric type — `u8 + u16`, `int + u32`, `i32 + float`, `number + u32` — is a compile-time error (`E0201`); there is no implicit conversion between fixed-bit types or between a fixed-bit type and the user-facing layer (except the literal coercion described above). In the user-facing layer the table is exhaustive: a numeric combination it does not give is a compile-time error (`E0201`).
- Comparison operators (`<`, `>`, `<=`, `>=`) follow the same operand rules and produce `bool`.

## null

`null` is both a literal — the keyword `null` — and a type: the type `null`, whose only value is `null`.

```xulo
let nothing: string? = null
let present: string? = "xulo"
let name = present ?? "stranger"
```

- `null` is a member of every optional type: for every type `T`, `null` is assignable to `T?`, and `T?` is exactly `T | null` (see [`composite-types.md`](composite-types.md)).
- Assigning `null` to a type that is not optional — `string`, `int`, `List<int>` — is a compile-time error (`E0201`). There is no implicit nullability.
- Null checking is strict. `null` is not a `bool` and is not interchangeable with `false`; a value of type `T?` MUST be tested against `null` or propagated with `??` and `?.` before it is used where `T` is required (see [`composite-types.md`](composite-types.md)).

## unit

`unit` is the type of expressions that produce no meaningful value. It has exactly one value, and the language provides no literal for that value — `unit` is never written `()` (see [`composite-types.md`](composite-types.md)). Values of the type are produced by expressions such as a call to a function that declares no return type or a `return` with no value (see [`../statements/return-and-block.md`](../statements/return-and-block.md)).

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
heterogeneous slot is written — `List<unknown>` holds values of any type, and
`Map<string, unknown>` maps strings to anything.

- Subtyping runs one way only: `T <: unknown` for every `T`, and `unknown <: T`
  only when `T` is `unknown`. Nothing else relates `unknown` to another type
  ([`type-relations.md`](type-relations.md)).
- An `unknown` value supports no operation except `==`/`!=`, the type test
  `is`, and `match`: member access on it is `E0102`, and using it as the
  operand of any other operator is `E0201`. The type records that a value
  exists, not what it is.
- `==` and `!=` accept an `unknown` operand, because `unknown` is the common
  type of any pair it joins, so `x == null` tests for absence. The test yields
  a `bool` like any other comparison and does **not** narrow `x`: a value of
  type `unknown` stays `unknown` in both branches, and neither does `x != null`.
- Inference never produces `unknown` on its own: a literal is `int`,
  `float`, `string`, `bool`, or `null`, or a type the context adapts it to.
  `unknown` appears only where it is written — an annotation, a type argument,
  a field type — or as the tested type of a `match` type pattern or of an `is`
  test.
- The ways to *use* such a value are `match` with a type pattern, which
  narrows it for that arm (`string s => …`), and `is`, which narrows the
  branches of an `if` (`data is string`)
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)).

## View

`View` is the built-in marker type produced by components: a component is a function whose declared return type is `View` (see [`../components/README.md`](../components/README.md)).

```xulo
fn Greeting(name: string): View {
  Text(`Hello, ${name}`)
}
```

Values of type `View` are opaque to the core type system beyond being renderable: apart from appearing as a component's result, as an element of a rendered `List<View>`, and as a child of another component, the language defines no field access, method call, indexing, or conversion on `View`. The concrete component vocabulary is provided by the UI library, not by this specification.

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

The names of the built-in composite types are reserved for the language. A module-scope `struct`, `enum`, `trait`, or `type` declaration MUST NOT use any of the following names:

`List`, `Array`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error`, `Ptr`

Declaring a type with one of these names at module scope is a compile-time error (`E0106`). The primitive value types need no entry in this list: their spellings are keywords, stopped at the lexical layer before a declaration is ever checked. `List`, `Array`, `Map`, and `Set` are specified in [`composite-types.md`](composite-types.md), `Task<T>` in [`function-types.md`](function-types.md), `Result` and `Error` in [`../error-handling.md`](../error-handling.md), and `View` in [`../components/view-syntax.md`](../components/view-syntax.md); `Range<T>` is specified above in this file. `Ptr` is specified in [`../ffi.md`](../ffi.md); it names an opaque foreign handle, not a value type, and is reserved for that reason. Elsewhere — as a type parameter name or an ordinary value binding — a built-in type name is an ordinary identifier, though using one conflicts with the naming conventions in [`../names.md`](../names.md).
