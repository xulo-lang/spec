# Control Flow — Delta: Map Values Through get

> **Delta** — replaces the iteration bullet in the `for` section of
> `expressions/control-flow.md`; the iterable table and the surrounding prose
> are unchanged.

- Iterating a `Map` visits its **keys**; the corresponding values are read
  with `m.get(k)` (see [`path-and-access.md`](path-and-access.md)). Iterating
  a `List`, `Map`, `Set`, or `Range` is core syntax; the other operations on
  the collections are inherent members
  (see [`../types/composite-types.md`](../types/composite-types.md)).
