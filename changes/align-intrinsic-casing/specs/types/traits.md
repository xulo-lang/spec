# Traits — Delta: Intrinsic Casing

> **Delta** — edits the `ToString` example and one `#Derive` bullet in
> `types/traits.md`.

## ToString example, replaced

```xulo
struct User { name: String, age: Int }

impl ToString for User {
  fn toString(self): String {
    `${self.name} (${self.age})`
  }
}
```

## `#Derive` — ToString bullet, edited

The bullet becomes:

- On a struct `S` with fields `f₁ … fₙ`, the generated `impl ToString for S`
  renders the value as `S(f1: …, f2: …)` — the fields in declaration order,
  each rendered by the rules `str` already uses: a base type as itself, a
  type implementing `ToString` through its `toString`. A struct with no
  fields renders as `S()`.
