# Operators — Delta: StringBuilder

> **Delta** — adds `StringBuilder` to the list of types with structural
> equality in `## Equality` of `expressions/operators.md`; the rest is
> unchanged.

Replace the equality paragraph with:

Equality is structural for all value types: primitives (`bool`, `string`,
`int`, `float`, fixed-bit numerics), `List`, `Array`, `Map`, `Set`,
`StringBuilder`, tuples, named `struct`s, and `enum`s — for an `enum`, the
variant and every payload value are compared; for a `StringBuilder`, the
current text is compared; for a tuple, the arities must be equal and every
pair of elements is compared, so different arities are a compile-time error
(`E0211`) rather than `false`; for an `Array`, elements are compared pairwise
in index order, and the length is part of the type, so arrays of different N
are unrelated types and their comparison is `E0211` rather than `false`. Both
operands MUST have a common type; comparing unrelated types is a compile-time
error (`E0211`), so Xulo has no cross-type equality. `null == null` is
`true`, and comparing `null` with a value requires that value's type to be
optional. Values that carry identity — rather than structural content — are
compared by identity, as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md).
