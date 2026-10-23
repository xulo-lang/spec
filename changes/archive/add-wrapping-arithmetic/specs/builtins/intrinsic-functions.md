# Intrinsic Functions — Delta: Integer Functions

> **Delta** — adds a new `### Integer functions` table in the `Math`
> namespace of `builtins/intrinsic-functions.md`, between the Basic
> functions table and the Formatting section, and appends two bullets to
> the Math bullet list after the `Math::abs` bullet; the tables, bullets,
> and example before and after those passages are unchanged. The complete
> signature index gains the eight matching rows, alphabetically placed.

## `Math` namespace

### Basic functions

(unchanged)

### Integer functions

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math::wrappingAdd` | `fn(a: int, b: int): int` | `a + b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrappingSub` | `fn(a: int, b: int): int` | `a − b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrappingMul` | `fn(a: int, b: int): int` | `a × b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::popCount` | `fn(x: int): int` | the number of 1 bits in `x` |
| `Math::countLeadingZeros` | `fn(x: int): int` | the number of leading 0 bits — 64 when `x` is 0 |
| `Math::countTrailingZeros` | `fn(x: int): int` | the number of trailing 0 bits — 64 when `x` is 0 |
| `Math::rotateLeft` | `fn(x: int, amount: u64): int` | `x` rotated left by `amount` modulo 64 |
| `Math::rotateRight` | `fn(x: int, amount: u64): int` | `x` rotated right by `amount` modulo 64 |

### Formatting

(unchanged)

(… the Math bullet list, ending as follows — the two new bullets append
after the `Math::abs` bullet and before the example block)

- `Math::abs`, `Math::min`, and `Math::max` are declared over `number`, so an
  `int` or `float` operand is accepted and a fixed-bit operand is not
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- The `wrapping*` functions never trap: their result is the unique `int`
  congruent to the given sum, difference, or product modulo 2⁶⁴, taken as
  two's complement, and the operator overflow rule does not apply to them
  ([`../types/primitive-types.md`](../types/primitive-types.md)).
- `Math::popCount`, `Math::countLeadingZeros`, and
  `Math::countTrailingZeros` count bits of the 64-bit two's-complement
  representation of their argument;
  `Math::rotateLeft` and `Math::rotateRight` rotate that same
  representation and take their `amount` modulo 64, so an amount of 0 or
  64 is the identity and every amount is in domain.

## Complete signature index

(unchanged except for the eight new rows, alphabetically placed)

| Name | Signature | Section |
|------|-----------|---------|
| `Math::cos` | `fn(x: float): float` | [`Math` namespace](#math-namespace) |
| `Math::countLeadingZeros` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::countTrailingZeros` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::E` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::PI` | `float` (constant) | [`Math` namespace](#math-namespace) |
| `Math::popCount` | `fn(x: int): int` | [`Math` namespace](#math-namespace) |
| `Math::pow` | `fn(base: float, exp: float): float` | [`Math` namespace](#math-namespace) |
| `Math::rotateLeft` | `fn(x: int, amount: u64): int` | [`Math` namespace](#math-namespace) |
| `Math::rotateRight` | `fn(x: int, amount: u64): int` | [`Math` namespace](#math-namespace) |
| `Math::round` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::trunc` | `fn(x: float): int` | [`Math` namespace](#math-namespace) |
| `Math::wrappingAdd` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
| `Math::wrappingMul` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
| `Math::wrappingSub` | `fn(a: int, b: int): int` | [`Math` namespace](#math-namespace) |
