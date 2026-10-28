# Xulo Language Specification — Delta: Const Parameter Terminology Entry

> **Delta** — adds the **const argument** and **const parameter** rows to the
> Terminology table in `README.md`, after **component** and before **copy**
> (the table is alphabetical); the rest is unchanged.

| **const argument** | A compile-time `int` value written in a generic argument list for a const parameter: an integer literal, a `const` binding (module-level or block-level), or a const parameter name. Negative values are written through a `const` binding. See [`types/generics.md`](types/generics.md). |
| **const parameter** | `const N: int` in a generic parameter list; a compile-time `int` value substituted at each instantiation. It is never a type and carries no runtime argument. See [`types/generics.md`](types/generics.md). |
