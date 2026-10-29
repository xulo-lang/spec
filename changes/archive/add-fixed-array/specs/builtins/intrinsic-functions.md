# Intrinsic Functions — Delta: Array Rendering

> **Delta** — adds the `Array<T, N>` row to the rendering table under `## Output`
> and adds `Array` to the no-string-form list under `## Conversion` in
> `builtins/intrinsic-functions.md`; the rest is unchanged.

Replace the `Set<T>` rendering row with:

| `Set<T>` | as a list, in an unspecified order |
| `Array<T, N>` | `[`, then the renderings of the elements in index order separated by `, `, then `]` — `[1, 2]` |

Replace the no-string-form sentence with:

- Any other operand is a compile-time error (`E0212`) — a `List`, an `Array`, a
  `Map`, a `Set`, an `enum` with no `impl ToString`, or `null`
  ([`../type-system/errors.md`](../type-system/errors.md)).
