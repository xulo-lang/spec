# Names — Delta: StringBuilder

> **Delta** — adds `StringBuilder` to the built-in non-primitive type names in
> `## Reserved and Contextual Names` of `names.md`; the rest is unchanged.

Replace the built-in type-name bullet with:

- **Built-in (non-primitive) type names** — `List`, `Array`, `Map`, `Set`,
  `StringBuilder`, `Task`, `Range`, `View`, `Result`, `Error` — and
  **intrinsic and namespace names** — `print`, `println`, `str`, `Math`,
  `Time`, `Task` (the namespace), and `ast` — SHOULD NOT be shadowed; this is
  a recommendation, not a restriction, and an inner binding may still shadow
  any prelude name. Intrinsics are described in the
  [builtins chapter](builtins/README.md).
