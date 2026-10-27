# Prelude — Delta: Array Type

> **Delta** — assumes `change-collection-members`; adds `Array` to the
> built-in types table, the reserved names, and the built-in-members
> paragraph of `builtins/prelude.md`; the rest is unchanged.

| Type | Description | Chapter |
|------|-------------|---------|
| `Array<T, N>` | inline fixed-length sequence of N elements; N is a const parameter, N ≥ 0 | [`../types/composite-types.md`](../types/composite-types.md) |

| Category | Names |
|----------|-------|
| Value and marker types | `List`, `Array`, `Map`, `Set`, `Task`, `Range`, `Result`, `View`, `Error` |

The built-in collections (`List`, `Array`, `Map`, `Set`) and `string` expose
inherent members, specified with their types: `List`, `Array`, `Map`, and
`Set` in [`../types/composite-types.md`](../types/composite-types.md), and
`string` in [`../types/primitive-types.md`](../types/primitive-types.md).
