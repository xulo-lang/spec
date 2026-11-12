# Prelude — Delta: StringBuilder

> **Delta** — adds `StringBuilder` to the built-in types table, the reserved
> names table, and the built-in-members paragraph of `builtins/prelude.md`;
> the rest is unchanged.

Add to the built-in types table, after the `Set<T>` row:

| `StringBuilder` | mutable, growable UTF-8 text buffer; materialized with `.build()` | [`../types/composite-types.md`](../types/composite-types.md) |

Replace the “Value and marker types” row with:

| Value and marker types | `List`, `Array`, `Map`, `Set`, `StringBuilder`, `Task`, `Range`, `Result`, `View`, `Error`, `Format` |

Replace the built-in-members paragraph with:

The built-in collections (`List`, `Array`, `Map`, `Set`), the text builder
`StringBuilder`, and `string` expose inherent members, specified with their
types: `List`, `Array`, `Map`, `Set`, and `StringBuilder` in
[`../types/composite-types.md`](../types/composite-types.md), and `string` in
[`../types/primitive-types.md`](../types/primitive-types.md). They are not
prelude operations: they have no unqualified spelling, no import introduces
them, and they are not module-scope names. A local binding named `len` or
`get` does not shadow `xs.len` or `m.get`; a static member is reached through
a type name, as in `Map::from_entries`.
