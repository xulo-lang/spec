# Generics — Delta: Const Parameters

> **Delta** — replaces the chapter's opening paragraph; replaces the
> introductory paragraph and the first, second, and trait-parameter bullets of
> `## Type Parameters`; inserts `## Const Parameters` after the
> `## Type Parameters` section, before `## Bounds`; replaces the bullets of
> `### Type applications`; replaces the bound-checking sentence; replaces the
> opening paragraph, the listed bullets, and the closing paragraph of
> `## Inference`; and replaces the opening paragraph and first two bullets of
> `## Instantiation and monomorphization`, together with the later bullets
> that describe type arguments. The rest of the chapter is unchanged.

Replace the opening paragraph with:

A *generic* declaration is parameterized by *generic parameters*: type
parameters, whose names stand for types chosen at each use, and const
parameters, whose names stand for compile-time `int` values chosen at each
use. This chapter defines where generic parameters may be declared, how type
parameters are bounded, how their arguments are determined at a call site,
and what a generic definition means once those arguments are known. The
built-in parameterized types are described in
[`primitive-types.md`](primitive-types.md) and
[`composite-types.md`](composite-types.md); the relations between the
resulting types are in [`type-relations.md`](type-relations.md).

Replace the introductory paragraph of `## Type Parameters` with:

A generic parameter list is written `<` … `>` immediately after the name of
the declaration. Functions, `struct`s, `enum`s, `type` aliases, and `impl`
blocks MAY all be parameterized by type parameters, const parameters, or
both.

Replace the first bullet of `## Type Parameters` with:

- A declaration has at most one **generic parameter list**, written directly
  after the declaration name: `<T>`, `<K, V>`, `<const N: int>`,
  `<T, const N: int>`. It is comma-separated and its names MUST be distinct —
  `<T, T>` and `<T, const T: int>` are ill-formed. Each name is in scope for
  the remainder of the declaration: the signature, the body, and — for
  `struct`, `enum`, `type` — every field, payload, and aliased type.

Replace the no-default bullet of `## Type Parameters` with:

- A parameter list has **no default arguments**: every argument of an
  application is written, or — for type parameters only — inferred; there is
  no `T = int` and no `N = 3` form. A const argument is never inferred
  (`E0816`).

Replace the trait-parameter bullet of `## Type Parameters` with:

- A `trait` is never parameterized. Its grammar accepts a generic parameter
  list only so that the attempt can be diagnosed, and declaring one — type or
  const — is a compile-time error (`E0810`; see [`traits.md`](traits.md)).
  Trait method signatures therefore never mention type or const parameters.

Insert after the `## Type Parameters` section, before `## Bounds`:

## Const Parameters

A generic parameter list MAY contain **const parameters** of type `int`,
written `const N: int`:

```xulo
fn count<const N: int>(): int { N }
fn repeat<const N: int>(value: int): int { N * value }
struct Tagged<const N: int, T> { value: T }
```

- `N` is a compile-time value of type `int`, not a type. It MAY appear in a
  generic argument, in a constant expression, and in an ordinary expression
  in the declaration's body; it is substituted at each instantiation.
- `int` is the only admitted const parameter type; any other type is `E0812`.
  Const parameters have no bounds and no default values. Type and const
  parameter names share one list and MUST be distinct; both are in scope
  throughout the declaration. By convention a const parameter name is a single
  uppercase letter (`N`, `M`), following [`../names.md`](../names.md); like
  every naming convention, this is RECOMMENDED, not enforced.
- A call to a function with const parameters requires the full generic
  argument list (`count<3>()`); omitting it is `E0816`. A type application
  such as `Tagged<3, string>` writes every parameter too, and a missing
  argument there is the ordinary `E0230`. Version 1 has no const argument
  inference and no partial written list. An `impl` is not called: its const
  parameters are determined by matching the implemented type, as
  `traits.md` specifies.
- A const argument is an integer literal, a `const` binding (module-level or
  block-level), or a const parameter name, written in the ordinary
  generic-argument position. The checker interprets an argument by the kind
  of the parameter in that position: a const argument supplied for a type
  parameter, or a type argument supplied for a const parameter, is `E0813`.
  Names are resolved in the value namespace at a const position and in the
  type namespace at a type position.
- A negative value is written through a `const` binding: `-1` is a unary
  minus, not an integer literal, and is therefore not a const argument.
  `const M = -1` followed by `Tagged<M, int>` is the form.
- Const arithmetic is not a type argument in version 1. `N + 1` does not
  parse as a type list; write a `const` binding for a computed value.
- A const argument is evaluated at compile time; type identity uses the
  value: `count<3>` and `count<M>` with `const M = 3` are the same
  instantiation; `count<3>` and `count<4>` differ.
- A const parameter is a value in the body; it never names a type and carries
  no runtime argument of its own.

Replace the bullets of `### Type applications` with:

- The number of generic arguments MUST equal the number of generic parameters
  the declaration names, type and const combined. `Map<string>` and
  `Pair<int, string, 3>` are ill-formed (`E0230`).
- A generic declaration's name MUST NOT appear without its arguments:
  `let xs: List = []` is ill-formed (`E0230`), not shorthand for anything.
  A non-generic name carries no argument list at all: `int<string>` is
  ill-formed (`E0230`) for the same reason.
- A type parameter is never applied (`E0230`); a const parameter is never
  applied either. A const argument is checked by kind exactly as at a call:
  an integer literal, a `const` binding, or a const parameter name evaluated
  to a compile-time `int`. A kind mismatch is `E0813`; a wrong argument count
  is `E0230`.

Replace the bound-checking sentence with:

**Bound checking.** Every type argument determined for a declaration MUST
satisfy every bound of the corresponding parameter — whether the argument was
inferred at a call site or written in a type application, and whether the
declaration is a function, a `struct`, an `enum`, an alias, or an `impl`. A
const argument has no bound.

Replace the opening paragraph of `## Inference` with:

A call MAY write the generic arguments of a generic callee explicitly — type
arguments and, for a const parameter, const arguments — in a list between the
callee and the argument list. The written arguments take the place of
inference for that call. A callee with const parameters MUST write the list
(`E0816`); a type-only callee MAY omit it:

Replace the example that follows with:

```xulo
let n = first<int>([1, 2, 3])      // T = int, written at the call
let s = first(["a", "b"])          // T = string, inferred as before
let c = count<3>()                 // N = 3, a const argument is always written
```

Replace the listed bullets of `## Inference` with:

- The list is written `<` … `>` immediately before the argument list,
  comma-separated with an optional trailing comma, and it is **all or
  nothing**: a call writes every generic parameter of its callee — type and
  const — or writes none. A call to a callee with a const parameter cannot
  omit the list, because a const argument is never inferred (`E0816`).
- The number of written arguments MUST equal the number of generic parameters
  the callee declares, type and const combined; a longer or shorter list is
  `E0808`.
- Each written type argument MUST satisfy every bound of its parameter,
  exactly as an inferred one must; a violation is `E0801`. A written const
  argument has no bound.
- When the list is written, inference is suppressed for the type parameters:
  no type variable is introduced for them, no unification constraint is
  raised against the value arguments, and an expected type MUST NOT override
  a written argument. A written const argument is used as given. A mismatch
  between a written argument and its parameter kind is `E0813`. The value
  arguments are then checked against the parameter types with the written
  type arguments substituted, and a mismatch is the ordinary argument error
  `E0202`:

  ```xulo
  let a = first<int>([1, 2, 3])      // OK: `List<int>` for `List<T>`
  let b = first<int>(["a", "b"])     // error[E0202]: expected `List<int>`, found `List<string>`
  ```

- The type of the call is the declared return type with the written type
  arguments substituted — each const parameter replaced by its evaluated
  value wherever it occurs in a type — and an expected type still checks that
  result:

  ```xulo
  let xs: int = first<int>([1, 2, 3])      // OK: the call has type int
  let ys: string = first<int>([1, 2, 3])   // error[E0201]: expected `string`, found `int`
  ```

- A callee that declares a const parameter and is called without a written
  list is `E0816`; const arguments are not inferred. Type-only callees keep
  the inference rules that follow.

Later in the same bullet list of `## Inference`, replace the `E0807` bullet
and the availability bullet with:

- A list written before a callee that declares no generic parameters is
  `E0807`.
- The list is available wherever a call is written — on a function, a method,
  a trait static, or a namespace member — whenever the callee declares
  generic parameters:

Replace the `Inference for an omitted list` paragraph with:

**Inference for an omitted list is unchanged** for a callee with only type
parameters: a call that writes no generic arguments to such a callee is
checked exactly as follows.

Replace the closing paragraph of `## Inference` with:

For type parameters, the written list is an additional way to supply what
inference would otherwise have to find; it adds no new freedom to a call that
omits it. For const parameters, the written list is the only way: const
arguments are never inferred.

Replace the opening paragraph and first two bullets of
`## Instantiation and monomorphization` with:

Each distinct assignment of type **and const** arguments identifies a
separate *instantiation*. The meaning of a program is defined by an as-if
rule: a generic declaration behaves as if a distinct, non-generic copy were
written for each instantiation, with every occurrence of each type parameter
replaced by its type and each const parameter by its integer value; that
value is a compile-time constant in the copy's body, not a runtime argument.
Instantiations are independent: using `first` at `int` and at `string` means
exactly what two hand-written functions `first_int` and `first_string` would
mean, and `Tagged<3, string>` differs from `Tagged<4, string>`.

Replace the later bullets of that section with:

- Wherever a type or const parameter occurs, distinct instantiations are
  distinct types: `Pair<int, string>` differs from `Pair<string, int>`, and
  `Tagged<3, string>` differs from `Tagged<4, string>`.
- No operation observes a type or const argument at runtime: no value
  carries one, no expression can request one, and neither a type pattern nor
  the right operand of `is` may name a type argument or a type parameter.
  Const arguments and const parameters are compile-time values; they do not
  become runtime arguments. There is therefore no observable difference
  between a program written with generics and the same program written with
  one concrete definition per instantiation. Diagnostics are correspondingly
  per-instantiation: an error in a generic body applies whenever the
  declaration is instantiated at a violating type.
