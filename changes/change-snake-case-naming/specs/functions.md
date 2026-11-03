# Functions — Delta: Snake-Case Official Names

> **Delta** — replaces the built-in-member sentence in “Methods and `self`”
> and the Static members bullet in `functions.md`.

## Methods and `self` (sentence)

- Methods are functions declared inside an `impl` block. The built-in
  collection and string types also declare **inherent members** specified
  with their types
  ([`types/composite-types.md`](types/composite-types.md),
  [`types/primitive-types.md`](types/primitive-types.md)); the methods among
  them are methods in the call and method-value sense, while `.len`, `.size`,
  and `.byte_len` are read-only properties. None is an `impl` declaration, and
  a program cannot add an inherent member to them; `impl Trait for` a built-in
  type is an ordinary trait implementation. The first parameter is
  the **receiver**, written `self` (an immutable borrow) or `mut self` (a
  mutable borrow). `self` is an explicit parameter, never implicit. A method
  that assigns to `self` or a field of `self` MUST declare `mut self`.

## Static members (bullet)

- **Static members.** `Map::from_entries(entries)`, `Set::from_list(values)`,
  and `Array::filled<T, const N: int>(value: T)` are associated members of the
  built-in types, reached through `::` like the associated names of an `impl`
  block. They have no receiver and are never selected by `.`.
