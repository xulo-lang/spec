# Functions — Delta: Array Static Member

> **Delta** — assumes `change-collection-members`; adds `Array::filled` to the
> static-members bullet of `## Methods and self` in `functions.md`; the rest is
> unchanged.

Replace the static-members bullet with:

- **Static members.** `Map::fromEntries(entries)`, `Set::fromList(values)`, and
  `Array::filled<T, const N: int>(value: T)` are associated members of the
  built-in types, reached through `::` like the associated names of an `impl`
  block. They have no receiver and are never selected by `.`.
