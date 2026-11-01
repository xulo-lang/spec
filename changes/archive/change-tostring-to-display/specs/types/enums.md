# Enums — Delta: String Form — Display, Debug, and Format

> **Delta** — renames the protocol in the interpolation bullet of
> `types/enums.md`. The rule is unchanged.

The bullet becomes:

- An enum value is never implicitly converted to `string`. Interpolation of an
  enum value requires an `impl Display for …` as described in
  [`traits.md`](traits.md).
