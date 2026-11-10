# Control Flow — Delta: Closed Range Operator

> **Delta** — replaces the range row of the pattern table, two example lines,
> and the Ranges section in `expressions/control-flow.md`.

## Patterns

The `Range` row of the pattern table becomes:

| Range | `a..<b`, `a..=b` | numeric values inside the half-open or closed range |

The range-pattern bullet becomes:

- The operands of a range pattern MUST be numeric literals of the same numeric
  type as the scrutinee. `0..=9` includes both endpoints, `0..<10` excludes the
  upper endpoint.

## Examples

The grade example becomes:

```xulo
let grade = match score {
  90..=100 => "A"
  80..<90 => "B"
  _ => "F"
}
```

## Ranges

The section becomes:

- `a..<b` is **half-open**: it includes `a` and excludes `b`.
- `a..=b` is **closed**: it includes both `a` and `b`.
- Both operands MUST have the same numeric type; the result is a value of the
  built-in generic type `Range<T>`, which MAY be bound and iterated later.
- Iterating a range yields `start`, `start + 1`, and so on while the value has
  not passed the end bound.

| Expression | Iteration values |
|------------|------------------|
| `0..<3` | `0`, `1`, `2` |
| `0..=3` | `0`, `1`, `2`, `3` |
| `1..=1` | `1` |
| `2..<2` | none |
| `5..<1` | none |
| `5..=1` | none |

- A range whose `start` is greater than its `end` iterates zero times. There is
  no reverse iteration: neither `..<` nor `..=` descends.
- `..<` and `..=` are non-associative operators at the range level of the
  precedence table in [`operators.md`](operators.md).
- Inside a list or map literal an element that starts with `...` is a spread
  of that literal ([`literals.md`](literals.md)); `...` is never a range
  operator.
