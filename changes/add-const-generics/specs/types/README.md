# Types — Delta: Generic Parameter Wording

> **Delta** — replaces the `Type parameter` row and the generic-inference
> bullet, replaces the `generics.md` row of `## How to Read This Chapter`,
> and adds a note under `## Type Kinds` in `types/README.md`; the rest is
> unchanged.

Replace the `Type parameter` row with:

| Generic parameter | type parameter `T`, `<T: Area>`; const parameter `const N: int` | [`generics.md`](generics.md) |

Replace the generic-inference bullet with:

- Type arguments are inferred at each call site; a call MAY instead write
  them explicitly, in a list between the callee and the argument list —
  `first<int>(…)`, `Map<string, int>(…)` — which takes the place of inference
  for that call. A const argument is never inferred and MUST be written
  (`count<3>()`); see [`generics.md`](generics.md).

Replace the `generics.md` row with:

- [`generics.md`](generics.md) — type and const parameters, bounds
  (`<T: Trait>`, `where` clauses), and call-site inference for type
  parameters; const arguments are explicit.

Add under the type-kinds table:

A const parameter is not a type kind: `const N: int` is a compile-time value
in a generic parameter list, not a type. See [`generics.md`](generics.md).
