# Control Flow — Delta: Iterating an Array

> **Delta** — adds `Array<T, N>` to the iterable table and the iteration bullet
> in the `for` section of `expressions/control-flow.md`; the rest is unchanged.
> Assumes `change-collection-members` (which rewrites the bullet).

| Iterable type | Loop binding | Iteration order |
|---------------|--------------|-----------------|
| `List<T>` | `T` | index order |
| `Array<T, N>` | `T` | index order |
| `Map<K, V>` | `K` | insertion order of keys |
| `Set<T>` | `T` | unspecified; programs MUST NOT depend on one |
| `Range<T>` | `T` | ascending from `start` |

Replace the iteration bullet with:

- Iterating a `Map` visits its **keys**; the corresponding values are read
  with `m.get(k)` (see [`path-and-access.md`](path-and-access.md)). Iterating
  a `List`, `Array`, `Map`, `Set`, or `Range` is core syntax; the other
  operations on the collections are inherent members
  (see [`../types/composite-types.md`](../types/composite-types.md)).
