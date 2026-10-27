# Generics — Delta: Const Parameters

> **Delta** — replaces the first “Type Parameters” bullet, inserts
> `## Const Parameters` after `## Type Parameters`, replaces the listed bullets
> of `## Inference`, and replaces the first two bullets of
> `## Instantiation and monomorphization`; the rest of `types/generics.md` is
> unchanged.

- A declaration has at most one **generic parameter list**, written directly
  after the declaration name: `<T>`, `<K, V>`, `<const N: int>`,
  `<T, const N: int>`. It is comma-separated and its names MUST be distinct —
  `<T, T>` and `<T, const T: int>` are ill-formed. Each name is in scope for
  the remainder of the declaration: the signature, the body, and — for
  `struct`, `enum`, `type` — every field, payload, and aliased type.

## Const Parameters

A generic parameter list MAY contain **const parameters** of type `int`,
written `const N: int`:

```xulo
fn count<const N: int>(): int { N }
fn repeat<const N: int>(value: int): int { N * value }
struct Tagged<const N: int, T> { value: T }
```

- `N` is a compile-time value of type `int`, not a type. It MAY appear in a
  type argument, in a constant expression, and in an ordinary expression in
  the declaration's body; it is substituted at each instantiation.
- `int` is the only admitted const parameter type; any other type is `E0812`.
  Const parameters have no bounds and no default values. Type and const
  parameter names share one list and MUST be distinct; both are in scope
  throughout the declaration.
- A declaration with const parameters requires the full generic argument
  list at every application (`count<3>()`, `Tagged<3, string>`). Version 1
  has no const argument inference and no partial written list.
- A const argument is an integer literal, a module `const` name, or a const
  parameter name, written in the ordinary type-argument position. The checker
  interprets an argument by the kind of the parameter in that position; a
  literal or const name at a type position, or a type at a const position, is
  `E0813`. Const arithmetic is not a type argument in version 1.
- A const argument is evaluated at compile time; type identity uses the value:
  `count<3>` and `count<M>` with `const M = 3` are the same instantiation;
  `count<3>` and `count<4>` differ.
- A const parameter is a value in the body; it never names a type and carries
  no runtime argument of its own.

**Inference.** The following bullets replace the all-or-nothing, count, and
const-inference rules of `## Inference`:

- The list is written `<` … `>` immediately before the argument list,
  comma-separated with an optional trailing comma, and it is **all or
  nothing**: a call writes every generic parameter of its callee — type and
  const — or writes none. A callee with a const parameter cannot omit the
  list, because a const argument is never inferred.
- The number of written arguments MUST equal the number of generic parameters
  the callee declares, type and const combined; a longer or shorter list is
  `E0808`.
- Each written type argument MUST satisfy every bound of its parameter,
  exactly as an inferred one must; a violation is `E0801`. A written const
  argument has no bound.
- When the list is written, type inference is suppressed for the type
  parameters; a written const argument is used as given. A mismatch between a
  written argument and its parameter kind is `E0813`.
- A callee that declares a const parameter and is called without a written
  list is an error; const arguments are not inferred.

**Instantiation and monomorphization.** The following replaces the first two
bullets of that section:

Each distinct assignment of type **and const** arguments identifies a
separate instantiation. The as-if rule replaces each type parameter by its
type and each const parameter by its integer value; that value is a
compile-time constant in the copy's body, not a runtime argument. Distinct
const values are distinct instantiations: `Tagged<3, string>` differs from
`Tagged<4, string>`.
