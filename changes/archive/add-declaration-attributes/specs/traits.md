# Traits — Delta: Declaration Attributes

> **Delta** — inserts a new **Attributes** section into `types/traits.md`,
> between **Structural vs nominal** and **Example**.

## Attributes

A declaration MAY carry attributes written directly before it: the name
`#Name`, optionally followed by a parenthesized, comma-separated list of
trait names with an optional trailing comma.

```xulo
#Derive(ToString)
struct User { name: String, age: Int }
```

- The attribute form is generic; the names the checker interprets form a
  **closed registry**. An attribute whose name is not in the registry is
  `E0108`. There are no inert attributes: every attribute written in source
  is interpreted by the checker.
- Attribute names occupy their own namespace. `#Derive` binds nothing and
  takes no name away from values, types, or modules.
- A registered attribute is valid only on the declarations its registry entry
  allows (`E0109`), and its arguments must fit the shape that entry states
  (`E0110`). Attributes are written on the five item declarations — `fn`,
  `struct`, `enum`, `trait`, and `impl` — and on nothing else
  ([`../grammar.md`](../grammar.md)).
- The registry of derive targets is closed in the same way. The only
  registered target is `ToString`. `Display` and `Debug` are reserved names
  in this registry and are not yet registered; written as targets they are
  `E0111`.

### Derive

`#Derive(ToString)` on a `struct` or an `enum` generates exactly the impl an
author would write by hand for the structural case. The body is fixed by the
target:

- On a struct `S` with fields `f₁ … fₙ`, the generated `impl ToString for S`
  renders the value as `S(f1: …, f2: …)` — the fields in declaration order,
  each rendered by the rules `str` already uses: a base type as itself, a
  type implementing `ToString` through its `to_string`. A struct with no
  fields renders as `S()`.
- On an enum, the generated body renders each variant exactly as the output
  table renders an enum value — `E::V`, or `E::V(p, …)` — with every payload
  value rendered by the field rules above. Payload slots are positional
  whether the variant declares them positionally or by name
  ([`../types/enums.md`](../types/enums.md)).

```xulo
#Derive(ToString)
struct User { name: String, age: Int }

#Derive(ToString)
enum Shape { Circle(Float), Rect(Float, Float) }

let s = User(name: "Alice", age: 30)
print(str(s))                    // User(name: Alice, age: 30)
print(`${Shape::Circle(2.5)}`)   // Shape::Circle(2.5)
```

Rules:

- A field or payload type that is neither a base type nor a type with a
  `ToString` impl has no string form in the generated interpolation; the
  derive fails with `E0801` at that type: `` `List<Int>` does not implement
  `ToString` ``.
- The expansion is an ordinary impl, so a hand-written `impl ToString` for
  the same type, or a second `#Derive(ToString)`, is the conflicting-impl
  error `E0805`.
- `#Derive` on a type that declares type parameters is `E0112`: the language
  has no bound that means "is a base type or implements `ToString`", so the
  expansion cannot be written for an open `T`. A generic type keeps its
  hand-written impl.
- A derived impl participates in dispatch exactly as a written one; derive
  introduces no satisfaction that an `impl` would not.
