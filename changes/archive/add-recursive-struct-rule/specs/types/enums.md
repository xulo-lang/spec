# Enums — Delta: Payload Recursion Permitted

> **Delta** — adds one bullet to the payload bullet list in `types/enums.md`,
> after the “A payload slot may have any type …” bullet; every other bullet
> is unchanged.

- The names of a named payload label the slots: they document the payload and
  appear in diagnostics, but they are not argument labels, because construction
  and matching are always positional. A payload slot may have any type —
  `struct`s, `enum`s, `List`s, `Map`s, and type parameters alike.
- A variant payload MAY mention the enum being declared, through other
  payloads included: `enum Tree { Leaf(int), Node(Tree, Tree) }` and
  `enum Step { Next(Step) }` are well-formed. Payload recursion needs no
  indirection; the field-composition cycle rule
  ([`composite-types.md`](composite-types.md), `E0113`) governs struct fields
  only.
