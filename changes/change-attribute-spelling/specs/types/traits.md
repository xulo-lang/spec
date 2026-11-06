# Traits — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the **Attributes**, **Derive**, and **Test** sections
> of `types/traits.md` with the lowercase bracket spelling. The registry rules, derive
> expansion, and test semantics are unchanged.

## Attributes

A declaration MAY carry attributes written directly before it: the form
`#[name]`, optionally with a parenthesized argument list inside the brackets;
what the arguments may contain is fixed per entry by the registry —
`#[derive]` takes a comma-separated list of trait names with an optional
trailing comma.

```xulo
#[derive(Debug)]
struct User { name: string, age: int }
```

An attribute on its own line, directly above the declaration it marks, is
**RECOMMENDED**; a shared line — `#[derive(Debug)] struct User { … }` — is
equally well-formed, the grammar being newline-insensitive
([lexical-structure.md](../lexical-structure.md)).

- The attribute form is generic; the names the checker interprets form a
  **closed registry**. An attribute whose name is not in the registry is
  `E0108`. There are no inert attributes: every attribute written in source
  is interpreted by the checker.
- Attribute names occupy their own namespace. `#[derive]` binds nothing and
  takes no name away from values, types, or modules.
- A registered attribute is valid only on the declarations its registry entry
  allows (`E0109`), and its arguments must fit the shape that entry states
  (`E0110`). Attributes are written on the five item declarations — `fn`,
  `struct`, `enum`, `trait`, and `impl` — and on the test form (`#[test]`),
  and on nothing else
  ([`../grammar.md`](../grammar.md)).
- The registry of derive targets is closed in the same way. The only
  registered target is `Debug`. `Display` and `ToString` are reserved names
  in this registry and are not registered; written as targets they are
  `E0111`. `Display` is never a derive target: the user-visible string form
  is an author's decision
  ([Built-in formatting traits](#built-in-formatting-traits)).

### Derive

`#[derive(Debug)]` on a `struct` or an `enum` generates exactly the impl an
author would write by hand for the structural case. The body is fixed by the
target:

- On a struct `S` with fields `f₁ … fₙ`, the generated `impl Debug for S`
  renders the value as `S(f1: …, f2: …)` — the fields in declaration order,
  each rendered by the field rules below: directly where no implementation
  is needed, through `debug` where one is. A struct with no fields renders
  as `S()`.
- On an enum, the generated body renders each variant exactly as the output
  table renders an enum value — `E::V`, or `E::V(p, …)` — with every payload
  value rendered by the field rules above. Payload slots are positional
  whether the variant declares them positionally or by name
  ([`../types/enums.md`](../types/enums.md)).
- The generated `debug` takes its `format` parameter like every `Debug`
  method: it forwards `format` to the `debug` calls it makes on
  user-declared fields and has no other use for it — the structural form is
  fixed. A body (generated or written) that never calls `debug` simply does
  not read the parameter, and a parameter is never reported as unused (the
  note under the warnings table of
  [`../type-system/errors.md`](../type-system/errors.md)).

```xulo
#[derive(Debug)]
struct User { name: string, age: int }

#[derive(Debug)]
enum Shape { Circle(float), Rect(float, float) }

let s = User(name: "Alice", age: 30)
print(s)                          // User(name: Alice, age: 30)
print(Shape::Circle(2.5))         // Shape::Circle(2.5)
```

Rules:

- The rendering of a field or payload is chosen by its type:
  - a `struct` renders through its `debug`; without a `Debug` impl the
    derive fails with `E0801` at that type: `` `User` does not implement
    `Debug` `` — a struct never renders directly;
  - an `enum` renders through its `debug` when it implements one, and
    otherwise by the output table's enum row, which needs no
    implementation — the same fallback `print` makes;
  - any other type renders directly — no implementation required **of the
    type itself** — by the output table's rules for its kind (a base type
    as itself, a container from its elements), the recursion below
    carrying these rules into every structural position that rendering
    reaches;
  - a value typed `unknown` — in a field or in any structural position —
    has no static rendering to expand: the derive fails with `E0212`, the
    error the same value's interpolation already gets.
- The rules are recursive through **structural positions** — a container's
  elements, an optional's `T`, a tuple's elements, a `Result`'s `T` and
  `E` slots, an enum's payloads (the term is defined with the output table
  ([`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md#output))) —
  and every position is rendered, and checked, by these same rules:
  `List<User>`, `User?`, `(User, int)`, and `Result<User, string>` each
  need `User`'s `Debug` and fail `E0801` at `User` without it. A wrapper is
  itself a user-declared type: `struct W { xs: List<int> }` derives because
  its field renders directly, and a `W` used as another derive's field
  needs `W`'s own `Debug`.
- The expansion is an ordinary impl, so a hand-written `impl Debug` for
  the same type, or a second `#[derive(Debug)]`, is the conflicting-impl
  error `E0805`.
- `#[derive]` on a type that declares generic parameters is `E0112`: the
  language has no bound that means "is not a `struct`, or implements
  `Debug`", so the expansion cannot be written for an open
  parameter. A generic type keeps its hand-written impl.
- A derived impl participates in dispatch exactly as a written one; derive
  introduces no satisfaction that an `impl` would not.
- `#[derive(Display)]` is `E0111`: user-visible text is written by hand, and
  the structural form a derive could default to is `Debug`'s.

### Test

`#[test]` marks a function as a test. It is spelled only by the test form —
`#[test]`, followed by `[async] fn name() { … }` — whose shape is fixed
([`../testing.md`](../testing.md)):

- the declaration has no parameters, no return type, no generic parameters,
  no `where` clause, and no `pub`;
- the attribute takes no arguments: `#[test()]`, `#[test("…")]`, and
  `#[test(name, …)]` do not form a `TestDecl`;
- the identifier binds no value, and any reference to it is `E0107`.

Written on a `struct`, `enum`, `trait`, or `impl`, `#[test]` is `E0109`
(`` attribute `test` is not valid on a struct ``). When the full test form
does not match, a `#[test]`, `#[test()]`, or `#[test(name, …)]` on the
non-matching `fn` — including a `fn` with parameters, a return type,
generics, a `where` clause, or `pub` — is `E0110`
(`` attribute `test` takes no arguments and is written `#[test]`, followed
by `[async] fn name() { … }` ``). A `#[test(...)]` group that the current
`AttributeArg` grammar cannot parse — notably a `StringLiteral` or a numeral
— is a syntax error when the full test form does not match; when
`add-macro-system` widens `AttributeArg`, such a group parses and falls under
the same `E0110` rule. When the sequence `#[test]` followed by `async`? `fn`
Identifier `(` `)` Block appears at file scope, the declaration is a
`TestDecl` and no registry lookup runs ([`../grammar.md`](../grammar.md)).
