# Control Flow — Delta: Iterating an Array

> **Delta** — adds `Array<T, N>` to the iterable table in the `for` section of
> `expressions/control-flow.md`; the rest is unchanged.

| Iterable type | Loop binding | Iteration order |
|---------------|--------------|-----------------|
| `List<T>` | `T` | index order |
| `Array<T, N>` | `T` | index order |
| `Map<K, V>` | `K` | insertion order of keys |
| `Set<T>` | `T` | unspecified; programs MUST NOT depend on one |
| `Range<T>` | `T` | ascending from `start` |
