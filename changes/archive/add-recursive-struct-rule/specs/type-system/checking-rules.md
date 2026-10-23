# Type Checking Rules — Delta: Value Composition

> **Delta** — adds one bullet after the “Type aliases” bullet in the
> declaration-checking list of `type-system/checking-rules.md`; the bullets
> before and after it are unchanged.

- **Type aliases.** Expanding aliases MUST terminate: an alias that expands,
  directly or through a chain, to itself is `E0105`; recursion through a type
  constructor is not a cycle.
- **Value composition.** A struct's field types MUST NOT reach the declaring
  struct through field positions, tuple elements, and alias expansion without
  an intervening indirection (`E0113`); recursion through `List<T>`,
  `Map<K, V>`, `Set<T>`, `T?`, a function type, or an enum payload is
  well-formed
  ([`../types/composite-types.md`](../types/composite-types.md)).
