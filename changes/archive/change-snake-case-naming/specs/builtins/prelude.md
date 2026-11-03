# Prelude — Delta: Snake-Case Official Names

> **Delta** — replaces the derived-action bullet under “Built-in protocols”,
> the `ToString` reserved-name paragraph, and the `Map::fromEntries`
> reference under “Built-in members” in `builtins/prelude.md`.

## Built-in protocols (bullet)

- `to_string` is the derived **action**, never written: every type that
  implements `Display` has `value.to_string()`, defined as `display` at the
  default `Format` — `Format(precision: null, alternate: false)` — and no
  `impl` declares the method.

## Reserved names (paragraph)

`ToString` stays reserved although it resolves to nothing as a trait: the
`to_string` method belongs to the language, and a module-scope `trait
ToString` would take the name while no rule uses it. Written where a trait
name is expected — `impl ToString for …` — it is an unresolved name
(`E0101`).

## Built-in members (paragraph)

The built-in collections (`List`, `Array`, `Map`, `Set`) and `string` expose
inherent members, specified with their types: `List`, `Array`, `Map`, and
`Set` in [`../types/composite-types.md`](../types/composite-types.md), and
`string` in [`../types/primitive-types.md`](../types/primitive-types.md). They
are not prelude operations: they have no unqualified spelling, no import
introduces them, and they are not module-scope names. A local binding named
`len` or `get` does not shadow `xs.len` or `m.get`; a static member is reached
through a type name, as in `Map::from_entries`.
