# Literals — Delta: Array Literals

> **Delta** — adds the array-literal paragraph after the list-literal rules in
> `expressions/literals.md`; the rest of the section is unchanged.

A list literal checked against an expected `Array<T, N>`
([`../types/composite-types.md`](../types/composite-types.md)) is an **array
literal**: its written element count MUST equal the constant N (`E0815`), and
its elements check against T in order; `[]` is the N = 0 case. An array
literal MUST NOT contain a spread — a spread has runtime length and cannot
satisfy a static N. Without an expected Array type, `[1, 2, 3]` remains a
`List<int>`; there is no separate array-literal syntax.
