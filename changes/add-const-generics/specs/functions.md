# Functions — Delta: Const Generic Functions

> **Delta** — replaces the generic-function bullet and the “Call-site type
> arguments” bullet in the `## Generic functions` section of `functions.md`;
> the rest is unchanged.

Replace the generic-function bullet with:

- A generic function declares a **generic parameter list** after its name —
  `<T, U, …>`, `<const N: int>`, or `<T, const N: int>`. Type parameters are
  types; const parameters are compile-time `int` values. Both are in scope
  over the signature and the body. A const parameter may be used in ordinary
  expressions and as a const argument; it is never a type.

Replace the “Call-site type arguments” bullet with:

- **Call-site generic arguments.** A call MAY write the generic arguments of
  a generic function explicitly, in a list between the callee and the
  argument list — `first<int>(…)`, `count<3>()` — or, for a callee with only
  type parameters, it MAY write none; there is no partial form. When the list
  is written it replaces inference for that call: its length MUST equal the
  declared generic-parameter count, type and const combined (`E0808`); every
  written type argument MUST satisfy its parameter's bounds (`E0801`); a
  written const argument MUST be an integer literal, a `const` binding, or a
  const parameter name (`E0813`), and is used as given. A callee that
  declares a const parameter and is called without a list is `E0816`; const
  arguments are never inferred. A list before a callee that declares no
  generic parameters is `E0807`. When the list is omitted for a type-only
  callee, type arguments are inferred as before.
