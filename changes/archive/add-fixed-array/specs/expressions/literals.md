# Literals — Delta: Array Literals

> **Delta** — qualifies the list-literal element-type sentence and adds the
> array-literal paragraph after the list-literal rules in
> `expressions/literals.md`; the rest of the section is unchanged.

Replace “The element type is the common type of the elements, so `[1, 2, 3]`
has type `List<int>`.” with:

Absent an expected `Array` type, the element type is the common type of the
elements, so `[1, 2, 3]` has type `List<int>`.

A list literal checked against an expected `Array<T, N>`
([`../types/composite-types.md`](../types/composite-types.md)) is an **array
literal**: its written element count MUST equal the constant N (`E0815`), and
its elements check against T in order; `[]` is the N = 0 case. The check
requires both sides to be known statically, so an array literal that contains
a spread (runtime length), or that is checked against an abstract const
parameter N in a generic declaration, is `E0818`. Without an expected Array
type, `[1, 2, 3]` remains a `List<int>`; there is no separate array-literal
syntax.
