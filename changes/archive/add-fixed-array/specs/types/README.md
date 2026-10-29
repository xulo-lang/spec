# Types — Delta: Array Type Kind

> **Delta** — adds `Array<T, N>` to the `Collection` row of the type-kinds table
> and adds `Array` to the `composite-types.md` bullet of
> `## How to Read This Chapter` in `types/README.md`; the rest is unchanged.

Replace the `Collection` row with:

| Collection | `List<T>`, `Array<T, N>`, `Map<K, V>`, `Set<T>` | [`composite-types.md`](composite-types.md) |

Replace the `composite-types.md` bullet of `## How to Read This Chapter` with:

- [`composite-types.md`](composite-types.md) — `List`, `Array`, `Map`, `Set`,
  positional tuples, nominal `struct` records, optional/union/intersection
  types, and type aliases.
