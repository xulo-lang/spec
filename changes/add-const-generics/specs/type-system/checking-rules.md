# Checking Rules — Delta: Const Generic Arguments

> **Delta** — adds the const-argument rule to the “Calls” section of
> `type-system/checking-rules.md`; the rest is unchanged.

- A written generic argument is checked against the callee's parameter kind.
  At a type parameter position it MUST be a type; at a const parameter position
  it MUST be an integer literal, a module `const` name, or a const parameter
  name, and it is evaluated to an `int`. A kind mismatch is `E0813`; a const
  parameter whose declared type is not `int` is `E0812`. Const arguments are
  explicit: there is no const inference. The written list covers type and
  const parameters together, and its length follows `E0808`.
