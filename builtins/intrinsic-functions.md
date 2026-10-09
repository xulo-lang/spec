# Intrinsic Functions

The prelude decides which names every module has; this chapter gives the
functions among them their signatures and their meaning. An intrinsic is a
function, or a member of a namespace, that the compiler provides directly:
it is available without import, it has exactly one signature, and no module
may redeclare its name. This chapter states the conventions those signatures
follow, specifies `print`, `println`, and `str`, and then the members of the
three built-in namespaces `Math`, `Time`, and `Task`. The names themselves are
in scope because of the prelude ([`prelude.md`](prelude.md)).

## Conventions

- A signature is written `name(params): result`, as in
  [`../functions.md`](../functions.md). A generic function writes its type
  parameters before the parameter list — `all<T>(tasks: List<Task<T>>): Task<List<T>>`
  — and they are inferred at the call site.
- Intrinsics are unqualified: `print(...)`, `str(...)`. A namespace member is
  written `Math.abs`, `Time.now`, `Task.all` — the namespace is an ordinary
  name, `.` selects the member, and the bare name (`abs`, `now`, `all`) is in
  scope nowhere ([`../expressions/path-and-access.md`](../expressions/path-and-access.md)).
- Every intrinsic has exactly one signature: there is no overloading, and
  neither default arguments nor named arguments apply to a call. The argument
  count and the argument types MUST match the signature
  ([`../functions.md`](../functions.md)).
- `print`, `println`, and `str` are intrinsics, not keywords
  ([`../lexical-structure.md`](../lexical-structure.md)); a module-scope
  declaration MUST NOT bear any intrinsic name
  ([`prelude.md`](prelude.md#reserved-names)).

## Output

| Signature | Result | Effect |
|-----------|--------|--------|
| `print<T>(value: T): Unit` | `Unit` | writes the rendering of `value` to standard output |
| `println<T>(value: T): Unit` | `Unit` | writes the rendering of `value` to standard output, then a line terminator |

- The operand MAY have **any type**: `print` and `println` are the only
  facilities that render a structured value directly, with no conversion step.
- `print` writes no line terminator of its own; `println` writes one after the
  rendering. The choice between them therefore decides whether the next output
  begins on a new line.
- Both return `Unit`, so a call appears as an expression statement and its
  value is never read ([`../statements/README.md`](../statements/README.md)).

The rendering of a value is computed recursively by this table; the rendering
of a container is built from the renderings of its elements.

| Value | Rendering |
|-------|-----------|
| `String` | the characters of the string itself, with no surrounding quotes and no escaping |
| `Boolean` | `true` or `false` |
| `Int` | the decimal numeral of the value |
| `Float` | the decimal numeral, with no fractional part when the value is whole (`3.0` renders as `3`); the special values render as `NaN`, `Infinity`, and `-Infinity` |
| `Null` | `null` |
| `List<T>` | `[`, then the renderings of the elements separated by `, `, then `]` — `[1, 2]` |
| `Map<K, V>` | `{`, then the entries in insertion order as `key: value` pairs separated by `, `, each side rendered by this table, then `}` — `{ name: "lyy", age: 30 }` |
| `Set<T>` | as a list, in an unspecified order |
| an `enum` value | `Enum::Variant`, or `Enum::Variant(p, …)` when the variant has payloads, each payload rendered by this table |
| a function, a `Task`, a `View` | `<function>`, `<task>`, `<view>` |

The order of a `Set` rendering is as unspecified as its iteration order.

```xulo
print("hi")                     // hi
print(1 + 2)                    // 3
print([1, 2])                   // [1, 2]
print(`count=${3}`)             // count=3

let user = { name: "lyy", age: 30 }
print(user)                     // { name: "lyy", age: 30 }
println("done")                 // done, then a line terminator
```

## Conversion

| Signature | Result |
|-----------|--------|
| `str(value: Int \| Float \| Boolean \| String \| ToString): String` | the string form of `value` |

- The operand MUST be a base type — `Int`, `Float`, `Boolean`, `String` — or a
  value whose type implements the built-in `ToString` protocol
  ([`prelude.md`](prelude.md#built-in-protocols)). In the signature above,
  `ToString` denotes the type of the values that implement the protocol.
- `str(value)` and the interpolation `${value}` accept exactly the same
  operands: where one is well-formed so is the other
  ([`../expressions/literals.md`](../expressions/literals.md)).
- A base type is rendered exactly as in the table under [Output](#output); for
  a `ToString` implementor the result is the value of `to_string` on the
  operand.
- Any other operand is a compile-time error — a `List`, a `Map`, a `Set`, an
  `enum` with no `impl ToString`, or `null`
  ([`../type-system/errors.md`](../type-system/errors.md)).

There is no implicit conversion anywhere else in the language: `+` concatenates
two strings and never converts its operands, so a mixed expression is written
with `str` ([`../expressions/operators.md`](../expressions/operators.md)).

```xulo
let name = "ada"
let n = 42
let label = "count=" + str(n)          // "count=42"
let who = `hello ${name}`              // interpolation needs no str
let maybe: Int? = null
let absent = str(maybe ?? 0)           // "0": narrow an optional first
let bad = str([1, 2])                  // error: List<Int> has no string form
```

## `Math` namespace

`Math` is a built-in namespace of mathematical constants and functions,
available without import. Its members are reached with `.` (`Math.PI`,
`Math.max(a, b)`), and its constants are ordinary `Float` values.

### Constants

| Member | Type | Value |
|--------|------|-------|
| `Math.PI` | `Float` | 3.141592653589793 |
| `Math.E` | `Float` | 2.718281828459045 |
| `Math.TAU` | `Float` | 6.283185307179586 |
| `Math.INFINITY` | `Float` | positive infinity |
| `Math.NAN` | `Float` | not a number |

### Basic functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math.abs` | `fn(x: Number): Number` | absolute value |
| `Math.min` | `fn(a: Number, b: Number): Number` | the smaller of two values |
| `Math.max` | `fn(a: Number, b: Number): Number` | the larger of two values |
| `Math.sqrt` | `fn(x: Float): Float` | square root |
| `Math.cbrt` | `fn(x: Float): Float` | cube root |
| `Math.pow` | `fn(base: Float, exp: Float): Float` | `base` raised to `exp` |
| `Math.floor` | `fn(x: Float): Int` | largest integer not above `x` |
| `Math.ceil` | `fn(x: Float): Int` | smallest integer not below `x` |
| `Math.round` | `fn(x: Float): Int` | nearest integer to `x` |
| `Math.trunc` | `fn(x: Float): Int` | integer formed by discarding the fractional part |

### Trigonometric functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math.sin` | `fn(x: Float): Float` | sine of an angle in radians |
| `Math.cos` | `fn(x: Float): Float` | cosine of an angle in radians |
| `Math.tan` | `fn(x: Float): Float` | tangent of an angle in radians |
| `Math.asin` | `fn(x: Float): Float` | arcsine, in radians |
| `Math.acos` | `fn(x: Float): Float` | arccosine, in radians |
| `Math.atan` | `fn(x: Float): Float` | arctangent, in radians |
| `Math.atan2` | `fn(y: Float, x: Float): Float` | arctangent of `y / x`, using the signs of both to choose the quadrant |

### Logarithmic and exponential functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math.log` | `fn(x: Float): Float` | natural logarithm |
| `Math.log2` | `fn(x: Float): Float` | base-2 logarithm |
| `Math.log10` | `fn(x: Float): Float` | base-10 logarithm |
| `Math.exp` | `fn(x: Float): Float` | `e` raised to `x` |

- Every `Float` parameter and result follows IEEE-754 binary64: an operation
  outside its domain yields `NaN`, an overflow yields the appropriate infinity,
  and the special values propagate through further operations
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- `Math.sqrt` of a negative value yields `NaN`; `Math.cbrt` is defined for
  every `Float`, negative values included.
- `Math.log`, `Math.log2`, and `Math.log10` yield `-Infinity` at `0` and `NaN`
  for a negative argument; `Math.exp` yields `Infinity` once its result
  overflows.
- `Math.asin` and `Math.acos` yield `NaN` outside `-1.0...1.0`.
- `Math.floor`, `Math.ceil`, and `Math.trunc` discard the fractional part
  toward negative infinity, toward positive infinity, and toward zero; they
  return `Int`. `Math.round` returns the nearest `Int`, and a value exactly
  halfway between two integers rounds away from zero.
- `Math.abs`, `Math.min`, and `Math.max` are declared over `Number`, so an
  `Int` or `Float` operand is accepted and a fixed-bit operand is not
  ([`../types/primitive-types.md`](../types/primitive-types.md)).

```xulo
let radius = 2.5
let circumference = 2.0 * Math.PI * radius
let a = 3.0
let b = 4.0
let hypotenuse = Math.sqrt(a * a + b * b)
let value = 120
let clamped = Math.min(Math.max(value, 0), 100)
let halfPi = Math.PI / 2
let whole = Math.floor(3.7)            // 3, of type Int
```

## `Time` namespace

`Time` is a built-in namespace of timestamps and async sleep, available without
import. Its functions return scalars only: `U64` for the clocks and a task for
the delay.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Time.now` | `fn(): U64` | current Unix time in milliseconds |
| `Time.now_nanos` | `fn(): U64` | current Unix time in nanoseconds |
| `Time.monotonic` | `fn(): U64` | current reading of the monotonic clock in milliseconds |
| `Time.monotonic_nanos` | `fn(): U64` | current reading of the monotonic clock in nanoseconds |
| `Time.sleep` | `fn(ms: U64): Task<Unit>` | delays for at least `ms` milliseconds |

- `Time.now` and `Time.now_nanos` read the wall clock: milliseconds and
  nanoseconds since the Unix epoch. They are appropriate for timestamps, not
  for measuring how long something took.
- `Time.monotonic` and `Time.monotonic_nanos` read a clock that never runs
  backwards and is unaffected by changes to the wall clock. Only differences
  between two readings are meaningful, which makes them the right choice for
  elapsed time, debounce windows, and benchmarks.
- `Time.sleep` produces a `Task<Unit>` that settles once the delay has elapsed.
  A `Time.sleep(...)` call MUST be awaited: it is written
  `await Time.sleep(ms)` inside an `async` body, where it suspends that body
  for at least `ms` milliseconds and yields `Unit`. `await` is legal only
  inside an `async` body
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).

```xulo
let start = Time.monotonic_nanos()
// ... work ...
let elapsed = Time.monotonic_nanos() - start

async fn later() {
  await Time.sleep(1000)
  println("1 second later")
}
```

## `Task` namespace

`Task` is the built-in namespace of task combinators. Its signatures are the
ones given in
[`../expressions/async-expressions.md`](../expressions/async-expressions.md),
which also specifies where `await` may appear and how a task is started.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Task.all` | `all<T>(tasks: List<Task<T>>): Task<List<T>>` | completes when every task has completed; results in argument order |
| `Task.race` | `race<T>(tasks: List<Task<T>>): Task<T>` | completes with the first task to complete |
| `Task.resolve` | `resolve<T>(value: T): Task<T>` | an already-completed task holding `value` |

`Task.all` and `Task.race` combine tasks that already exist: neither starts any
work of its own, because a call to an `async` function has already started its
body ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
`Task.all` settles when every task of the list has settled and yields the
results in argument order; `Task.race` settles as soon as the first task of the
list settles and yields that task's result. `Task.resolve` wraps a value that
is already at hand, producing a task that has already settled: awaiting it
yields the value immediately. Generic type arguments are inferred at the call
site, and how results and cancellation combine through the combined tasks is
specified in [`../error-handling.md`](../error-handling.md).

```xulo
async fn fetchAll(): List<User> {
  let a = fetchUser(1)
  let b = fetchUser(2)
  await Task.all([a, b])            // the two results, in order
}

async fn fastest(a: Task<Int>, b: Task<Int>): Int {
  await Task.race([a, b])           // whichever settles first
}

async fn cached(): Int {
  let ready: Task<Int> = Task.resolve(42)
  await ready                       // 42, without suspending
}
```

## Complete signature index

Every intrinsic specified in this file, in alphabetical order by name.
Constants give their type in place of a signature.

| Name | Signature | Section |
|------|-----------|---------|
| `Math.abs` | `fn(x: Number): Number` | [`Math` namespace](#math-namespace) |
| `Math.acos` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.asin` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.atan` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.atan2` | `fn(y: Float, x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.cbrt` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.ceil` | `fn(x: Float): Int` | [`Math` namespace](#math-namespace) |
| `Math.cos` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.E` | `Float` (constant) | [`Math` namespace](#math-namespace) |
| `Math.exp` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.floor` | `fn(x: Float): Int` | [`Math` namespace](#math-namespace) |
| `Math.INFINITY` | `Float` (constant) | [`Math` namespace](#math-namespace) |
| `Math.log` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.log10` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.log2` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.max` | `fn(a: Number, b: Number): Number` | [`Math` namespace](#math-namespace) |
| `Math.min` | `fn(a: Number, b: Number): Number` | [`Math` namespace](#math-namespace) |
| `Math.NAN` | `Float` (constant) | [`Math` namespace](#math-namespace) |
| `Math.PI` | `Float` (constant) | [`Math` namespace](#math-namespace) |
| `Math.pow` | `fn(base: Float, exp: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.round` | `fn(x: Float): Int` | [`Math` namespace](#math-namespace) |
| `Math.sin` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.sqrt` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.tan` | `fn(x: Float): Float` | [`Math` namespace](#math-namespace) |
| `Math.TAU` | `Float` (constant) | [`Math` namespace](#math-namespace) |
| `Math.trunc` | `fn(x: Float): Int` | [`Math` namespace](#math-namespace) |
| `print` | `print<T>(value: T): Unit` | [Output](#output) |
| `println` | `println<T>(value: T): Unit` | [Output](#output) |
| `str` | `str(value: Int \| Float \| Boolean \| String \| ToString): String` | [Conversion](#conversion) |
| `Task.all` | `all<T>(tasks: List<Task<T>>): Task<List<T>>` | [`Task` namespace](#task-namespace) |
| `Task.race` | `race<T>(tasks: List<Task<T>>): Task<T>` | [`Task` namespace](#task-namespace) |
| `Task.resolve` | `resolve<T>(value: T): Task<T>` | [`Task` namespace](#task-namespace) |
| `Time.monotonic` | `fn(): U64` | [`Time` namespace](#time-namespace) |
| `Time.monotonic_nanos` | `fn(): U64` | [`Time` namespace](#time-namespace) |
| `Time.now` | `fn(): U64` | [`Time` namespace](#time-namespace) |
| `Time.now_nanos` | `fn(): U64` | [`Time` namespace](#time-namespace) |
| `Time.sleep` | `fn(ms: U64): Task<Unit>` | [`Time` namespace](#time-namespace) |
