# View Syntax — Delta: Iterating an Array in a Block

> **Delta** — adds `Array<T, N>` to the component-block `for` iterable in
> `components/view-syntax.md`; the rest is unchanged.

Replace the `for`-iterable sentence with:

- A `for` in a block renders one child per element. The iterable MUST be a
  `List<T>`, an `Array<T, N>`, a `Map<K, V>` (visiting its keys), a `Set<T>`,
  or a `Range<T>`; the loop variable is a fresh binding for each iteration, and
  the body's block is evaluated once per element, contributing its items in
  order.
