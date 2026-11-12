# Operators — Delta: Closed Range Operator

> **Delta** — replaces the range row of the precedence table, the spread note,
> and the Ranges section in `expressions/operators.md`.

## Precedence

The `range` row becomes:

| range | `..<` `..=` | non-assoc |

The note beneath the table becomes:

Prefix spread `...expr` exists only inside list and map literals; it is not
part of this chain. The closed-range operator is `..=`, a distinct infix
token; an element that begins with `...` is a spread.

## Ranges

The section becomes:

`a..<b` builds a half-open range that excludes `b`; `a..=b` builds a closed
range that includes `b`.

```xulo
for i in 0..<10 {
  print(i)
}
let slice = lo..=hi
```

Both operators are non-associative and yield the built-in generic type
`Range<T>`. The operands MUST have the same numeric type, and `T` is that type:
`0..<n` with `n: int` is `Range<int>` and `0.0..<x` with `x: float` is
`Range<float>`. Mixed operand types are not promoted, so `0..<x` with
`x: float` is an error (`E0201`) and MUST be written `0.0..<x`.

A `Range<T>` is the operand of `for … in` iteration and is usable as a range
pattern in `match`; both are specified in
[`control-flow.md`](control-flow.md). See
[`../types/primitive-types.md`](../types/primitive-types.md).
