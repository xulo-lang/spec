# Traits — Delta: String Form — Display, Debug, and Format

> **Delta** — replaces the `ToString` bullet of the "Structural vs nominal"
> section, inserts a "Built-in formatting traits" section before
> "Attributes", adds the derived-`toString` paragraph to "Member calls",
> and replaces the `#Derive` examples, the derive-registry bullet, and the
> whole "Derive" section of `types/traits.md`.

## Structural vs nominal

The final bullet of the section — and the `ToString` example beneath it —
becomes:

- `Display`, `Debug`, and `Format` are built-in; no declaration introduces
  them, and the string-form rule, the derived `toString`, and the formatting
  contracts are stated in
  [Built-in formatting traits](#built-in-formatting-traits).

Insert the following section between "Structural vs nominal" and
"Attributes", in full:

## Built-in formatting traits

`Display`, `Debug`, and the struct `Format` are built-in: the prelude
provides them, no declaration introduces them, and every module sees them
([`../builtins/prelude.md`](../builtins/prelude.md#built-in-protocols)).
Their declarations, as the prelude states them:

```xulo
struct Format {
  precision: int?,   // the precision a directive asked for; `null` = none
                     // asked — `0` is a request, not an absence
  alternate: bool    // the `#` alternate form; `false` = not asked
}

trait Display {
  fn display(self, format: Format): string
}

trait Debug {
  fn debug(self, format: Format): string
}
```

- `Display` is **how a type is shown**: `display` returns the receiver's
  string form at the `Format` it is given. Every consumer that renders with
  no directive — interpolation in `` `…${…}…` ``, the intrinsic `str` (and
  a member that spells its `str` domain, `List.join`), `print`, and the
  derived `toString` — passes the **default** `Format`,
  `Format(precision: null, alternate: false)`.
- `Format` carries **semantics only**. It has no width, alignment, fill, or
  padding field: layout is applied by the consumer to the string `display`
  returns, never by an implementation, so a rendering request is honored
  whether or not the implementation looked at `format`. A `display` that has
  a meaning for `format.precision` or `format.alternate` SHOULD reflect it;
  ignoring either still leaves a well-formed rendering, because whatever the
  type did not lay out, the consumer does. A consumer hands a directive's
  precision to `display` unchanged: the contract fixes the field, not its
  unit — what the number counts (decimal places, significant digits,
  characters) is the implementation's reading of its own type.
- The field set may grow, but only by fields of an optional type: a `T?`
  field MAY be omitted at construction and takes `null`
  ([`../types/composite-types.md`](../types/composite-types.md)), so a
  construction written against the present fields keeps compiling and keeps
  its meaning — an omitted field always reads as "the consumer asked for
  nothing", the default rendering. A future boolean flag is written `bool?`
  for the same reason: the contract declares its `null` and its `false` one
  reading — *not asked* — because a non-optional field would fail every
  existing construction (`E0205`).
- `Debug` is **what a type shows of itself**: `debug` returns the structural
  form — the value's internals rendered the way the output table renders
  structure
  ([`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md#output)).
  The structural form does not read `format` in this version; the parameter
  is part of the signature so that a later rendering can.
- Every `Display` impl makes the **action** available: `value.toString()` is
  `value.display` at the default `Format`. The language provides the method
  with the impl — it dispatches through that impl, is resolved wherever the
  impl is visible (see [Member calls](#member-calls)), and is declared by
  no impl: an `impl Display` that declares
  `fn toString` is `E0804`, because `Display` declares no such method.
- A value interpolated by `${…}`, passed to `str`, or rendered through a
  member that spells the `str` domain (`List.join`'s elements) MUST be a
  base type (`int`, `float`, `bool`, `string`) or a type that implements
  `Display`; other types are a compile-time error in those positions
  (`E0212`).

  ```xulo
  struct User { name: string, age: int }

  impl Display for User {
    fn display(self, format: Format): string {
      `${self.name} (${self.age})`
    }
  }
  ```

In the "Member calls" section, this paragraph joins the resolution rules,
after the ambiguity paragraph:

Beyond the two steps — when neither finds a candidate — a receiver whose
type implements `Display`, through its own type or through a `T: Display`
bound, resolves `toString(…)` to the derived action of that impl: the
member is found without appearing in any trait's declared members,
dispatches to `display` at the default `Format`, and yields to anything
either step found (an inherent `toString` wins at step 1, one declared by
another in-scope trait at step 2). An `impl Display` that declares
`fn toString` remains `E0804`: the action is provided, not declared.

## Attributes

Three passages of the "Attributes" section change.

The example becomes:

```xulo
#Derive(Debug)
struct User { name: string, age: int }
```

The shared-line sentence becomes:

An attribute on its own line, directly above the declaration it marks, is
**RECOMMENDED**; a shared line — `#Derive(Debug) struct User { … }` — is
equally well-formed, the grammar being newline-insensitive
([lexical-structure.md](../lexical-structure.md)).

The final bullet becomes:

- The registry of derive targets is closed in the same way. The only
  registered target is `Debug`. `Display` and `ToString` are reserved names
  in this registry and are not registered; written as targets they are
  `E0111`. `Display` is never a derive target: the user-visible string form
  is an author's decision
  ([Built-in formatting traits](#built-in-formatting-traits)).

The whole "### Derive" section becomes:

### Derive

`#Derive(Debug)` on a `struct` or an `enum` generates exactly the impl an
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
#Derive(Debug)
struct User { name: string, age: int }

#Derive(Debug)
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
  the same type, or a second `#Derive(Debug)`, is the conflicting-impl
  error `E0805`.
- `#Derive` on a type that declares generic parameters is `E0112`: the
  language has no bound that means "is not a `struct`, or implements
  `Debug`", so the expansion cannot be written for an open
  parameter. A generic type keeps its hand-written impl.
- A derived impl participates in dispatch exactly as a written one; derive
  introduces no satisfaction that an `impl` would not.
- `#Derive(Display)` is `E0111`: user-visible text is written by hand, and
  the structural form a derive could default to is `Debug`'s.
