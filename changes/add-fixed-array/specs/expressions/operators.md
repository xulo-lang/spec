# Operators — Delta: Array Operands

> **Delta** — adds `Array<T, N>` to the structural-equality list and rules its
> operand behaviour for `+` and the arithmetic operators in
> `expressions/operators.md`; the rest of the chapter is unchanged.

Replace the first paragraph of the `## Equality` section with:

Equality is structural for all value types: primitives (`bool`, `string`,
`int`, `float`, fixed-bit numerics), `List`, `Array`, `Map`, `Set`, tuples,
named `struct`s, and `enum`s — for an `enum`, the variant and every payload
value are compared; for a tuple, the arities must be equal and every pair of
elements is compared, so different arities are a compile-time error (`E0211`)
rather than `false`; for an `Array`, elements are compared pairwise in index
order, and the length is part of the type, so arrays of different N are
unrelated types and their comparison is `E0211` rather than `false`. Both
operands MUST have a common type; comparing unrelated types is a compile-time
error (`E0211`), so Xulo has no cross-type equality. `null == null` is
`true`, and comparing `null` with a value requires that value's type to be
optional. Values that carry identity — rather than structural content — are
compared by identity, as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md).

In the `## string and List +` section, after the `List<T> + List<U>` rule,
add:

`Array` is not an operand of `+` or of any arithmetic operator: `Array +
Array`, `Array + List`, and `arr * 2` are compile-time errors (`E0201`; a
`string` mixed with an `Array` stays `E0227`). There is no Array
concatenation — the length of the result would be a const sum, and this
change adds no const arithmetic. Convert first, for example with `.slice()`,
which yields a `List`.
