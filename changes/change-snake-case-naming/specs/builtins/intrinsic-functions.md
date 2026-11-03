# Intrinsic Functions — Delta: Snake-Case Names

> **Delta** — replaces the `toString` reference under “Conversion”, the
> Integer, Formatting, and `Time` tables, the prose and examples that quote
> those members, and the Complete signature index in
> `builtins/intrinsic-functions.md`.

## Conversion (bullet)

- A base type is rendered exactly as in the table under [Output](#output); for
  a `Display` implementor the result is the value of `to_string` on the
  operand — its `display` at the default `Format`.

## Integer functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math::wrapping_add` | `fn(a: int, b: int): int` | `a + b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrapping_sub` | `fn(a: int, b: int): int` | `a − b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrapping_mul` | `fn(a: int, b: int): int` | `a × b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::pop_count` | `fn(x: int): int` | the number of 1 bits in `x` |
| `Math::count_leading_zeros` | `fn(x: int): int` | the number of leading 0 bits — 64 when `x` is 0 |
| `Math::count_trailing_zeros` | `fn(x: int): int` | the number of trailing 0 bits — 64 when `x` is 0 |
| `Math::rotate_left` | `fn(x: int, amount: u64): int` | `x` rotated left by `amount` modulo 64 |
| `Math::rotate_right` | `fn(x: int, amount: u64): int` | `x` rotated right by `amount` modulo 64 |

## Integer functions (prose bullets)

- The `wrapping_*` functions never trap: their result is the unique `int`
  congruent to the given sum, difference, or product modulo 2⁶⁴, taken as
  two's complement, and the operator overflow rule does not apply to them
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- `Math::pop_count`, `Math::count_leading_zeros`, and
  `Math::count_trailing_zeros` count bits of the 64-bit two's-complement
  representation of their argument;
  `Math::rotate_left` and `Math::rotate_right` rotate that same
  representation and take their `amount` modulo 64, so an amount of 0 or
  64 is the identity and every amount is in domain.

## Formatting

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math::to_fixed` | `fn(x: float, digits: int): string` | the decimal numeral of `x` with exactly `digits` digits after the decimal point |

## Formatting (prose bullet)

- `Math::to_fixed` rounds at the `digits`-th fractional place using the same
  halfway-away-from-zero rule, keeps trailing zeros — a fixed width is the
  point of the function — and renders the sign exactly as `str` does, so a
  negative zero yields a leading `-`. The special values ignore `digits` and
  render as `NaN`, `Infinity`, and `-Infinity`. A `digits` value outside
  `0...=15` is a runtime failure
  ([`../memory-and-runtime.md`](../memory-and-runtime.md)).

## `Math` namespace (example)

```xulo
let radius = 2.5
let circumference = 2.0 * Math::PI * radius
let a = 3.0
let b = 4.0
let hypotenuse = Math::sqrt(a * a + b * b)
let value = 120
let clamped = Math::min(Math::max(value, 0), 100)
let half_pi = Math::PI / 2
let whole = Math::floor(3.7)            // 3, of type int
print(`price: ${Math::to_fixed(3.14159, 2)}`)  // price: 3.14
print(`${Math::to_fixed(2.5, 0)}`)             // 3
print(`${Math::to_fixed(-2.5, 0)}`)            // -3
print(`${Math::to_fixed(3.1, 2)}`)             // 3.10
```

## `Time` namespace

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Time::now` | `fn(): u64` | current Unix time in milliseconds |
| `Time::now_nanos` | `fn(): u64` | current Unix time in nanoseconds |
| `Time::monotonic` | `fn(): u64` | current reading of the monotonic clock in milliseconds |
| `Time::monotonic_nanos` | `fn(): u64` | current reading of the monotonic clock in nanoseconds |
| `Time::sleep` | `fn(ms: u64): Task<unit>` | delays for at least `ms` milliseconds |

## `Time` namespace (prose bullets)

- `Time::now` and `Time::now_nanos` read the wall clock: milliseconds and
  nanoseconds since the Unix epoch. They are appropriate for timestamps, not
  for measuring how long something took.
- `Time::monotonic` and `Time::monotonic_nanos` read a clock that never runs
  backwards and is unaffected by changes to the wall clock. Only differences
  between two readings are meaningful, which makes them the right choice for
  elapsed time, debounce windows, and benchmarks.

## `Time` namespace (example)

```xulo
let start = Time::monotonic_nanos()
// ... work ...
let elapsed = Time::monotonic_nanos() - start

async fn later() {
  await Time::sleep(1000)
  println("1 second later")
}
```

## Complete signature index

Every intrinsic specified in this file, in alphabetical order by name,
ignoring case and underscores. Constants give their type in place of a
signature.

| Name | Signature | Section |
|------|-----------|---------|
| `assert` | `assert(condition: bool, message: string): unit` | [Assertion](#assertion) |
| `Math::abs` | `fn(x: number): number` | [`Math` namespace](#math-namespace) |
| `Math::acos` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::asin` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::atan` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::atan2` | `fn(y: float, x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::cbrt` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::ceil` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::cos` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::count_leading_zeros` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::count_trailing_zeros` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::E` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::exp` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::floor` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::INFINITY` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::log` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::log10` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::log2` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::max` | `fn(a: number, b: number): number` | [`Math` namespace](#math-namespace) |
| `Math::min` | `fn(a: number, b: number): number` | [`Math` namespace](#math-namespace) |
| `Math::NAN` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::PI` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::pop_count` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::pow` | `fn(base: float, exp: float): float` | [`Math` namespace](#math-namespace) |
| `Math::rotate_left` | `fn(x: int, amount: u64): int` | [`Math` namespace](#math-namespace) |
| `Math::rotate_right` | `fn(x: int, amount: u64): int` | [`Math` namespace](#math-namespace) |
| `Math::round` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::sin` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::sqrt` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::tan` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::TAU` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::to_fixed` | `fn(x: float, digits: int): string` | [`Math` namespace](#math-namespace) |
| `Math::trunc` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::wrapping_add` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
| `Math::wrapping_mul` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
| `Math::wrapping_sub` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
| `print` | `print<T>(value: T): unit` | [Output](#output) |
| `println` | `println<T>(value: T): unit` | [Output](#output) |
| `str` | `str(value: int \| float \| bool \| string \| Display): string` | [Conversion](#conversion) |
| `Task::all` | `all<T>(tasks: List<Task<T>>): Task<List<T>>` | [`Task` namespace](#task-namespace) |
| `Task::race` | `race<T>(tasks: List<Task<T>>): Task<T>` | [`Task` namespace](#task-namespace) |
| `Task::resolve` | `resolve<T>(value: T): Task<T>` | [`Task` namespace](#task-namespace) |
| `Task::timeout` | `timeout<T>(t: Task<T>, ms: u64): Task<Result<T, Error>>` | [`Task` namespace](#task-namespace) |
| `Time::monotonic` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::monotonic_nanos` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::now` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::now_nanos` | `fn(): u64` | [`Time` namespace](#time-namespace) |
| `Time::sleep` | `fn(ms: u64): Task<unit>` | [`Time` namespace](#time-namespace) |
