# Type Relations — Delta: Const Generic Arguments

> **Delta** — replaces the “Generic applications” bullet of `## Type Equality`
> and the generic-argument sentence of “Constraint generation and solving” in
> `types/type-relations.md`; the rest is unchanged.

Replace the “Generic applications” bullet with:

- **Generic applications.** `C<A₁, …, Aₙ>` and `C<B₁, …, Bₙ>` are equal exactly
  when the heads `C` are the same declaration and each argument is equal: a
  type argument when `Aᵢ = Bᵢ`, and a const argument when its compile-time
  `int` value is equal. So `Tagged<3, string>` and `Tagged<M, string>` with
  `const M = 3` are the same type, while `Tagged<3, string>` and
  `Tagged<4, string>` are different types. Different arguments give different
  types: `List<int>` and `List<string>` are unrelated, as are `Pair<int, int>`
  and `Pair<int, string>`.

In “Constraint generation and solving”, replace “a generic argument against the
type parameter it instantiates” with:

a generic argument against the parameter it instantiates — a type argument
against its type parameter, or a const argument against its const parameter's
`int` value.
