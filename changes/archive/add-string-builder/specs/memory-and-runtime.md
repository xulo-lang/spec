# Memory and Runtime — Delta: StringBuilder

> **Delta** — adds `StringBuilder` to the reference row of the
> value/reference table, the structural equality paragraph, and the `copy`
> bullet in `memory-and-runtime.md`; the rest is unchanged.

Replace the Reference row with:

| **Reference** | `List<T>`, `Map<K, V>`, `Set<T>`, `StringBuilder` | a reference to one shared value | the same value, shared: `x` and `e` then denote the one underlying value |

Replace the opening sentence of the equality paragraph with:

`==` compares by content for every value-carrying type — primitives,
`string`, `struct`, `enum`, `Range`, `List`, `Array`, `Map`, `Set`,
`StringBuilder`, and tuples (element-wise, equal arity) — as specified in
[`expressions/operators.md`](expressions/operators.md).

Replace the first `copy` bullet with:

- Deep copying follows references through `List`, `Map`, `Set`, and
  `StringBuilder` values, through `struct` and `enum` fields, through tuple
  elements, and through `Array` elements, and duplicates their contents.
  Copying a cyclic structure terminates: a value already being copied is
  represented by the copy made for it, so the copy graph mirrors the
  original.
