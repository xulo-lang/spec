# Type Relations — Delta: Array Variance

> **Delta** — adds the `Array<T, N>` row to the variance table and the Array
> subtyping note, and adds `Array` to the built-in-generics sentence in
> `types/type-relations.md`; the rest is unchanged.

| Constructor | Variance | Reason |
|-------------|----------|--------|
| `Array<T, N>` | invariant in `T`; N exact | elements are written through a subscript, and N is a const parameter |

- `Array<T, N> <: Array<U, M>` holds exactly when `T = U` and `N = M`; there is
  no subtyping between different lengths and no conversion to or from `List`.

Replace the built-in-generics bullet with:

- **Built-in generics.** Whether subtyping lifts through `List<T>`, `Array<T, N>`,
  `Map<K, V>`, `Set<T>`, `struct`s, `enum`s, and tuples is decided by their
  variance, below.
