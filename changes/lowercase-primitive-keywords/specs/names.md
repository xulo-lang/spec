# Names — Delta: Lowercase Primitive Keywords

> **Delta** — rewrites the two-groups bullet and the shadowing paragraph in
> the **Naming Conventions** section of `names.md`.

The two-groups passage becomes:

Two groups of names are built in rather than declared:

- **Primitive type keywords** — `bool`, `string`, `number`, `int`,
  `float`, `i8`…`u64`, `f32`, `f64`, `null`, `unit`, `unknown` — are
  keywords: no declaration MAY bear their spelling, and there is nothing to
  shadow.
- **Built-in (non-primitive) type names** — `List`, `Map`, `Set`, `Task`,
  `Range`, `View`, `Result`, `Error` — and **intrinsic names** — `print`,
  `println`, `str`, `Math`, `Time` (and the namespace `Task`) — SHOULD NOT
  be shadowed. Intrinsics are described in the
  [builtins chapter](builtins/README.md).

The shadowing paragraph becomes:

Shadowing a built-in type name means precisely this: a declaration
introduces an ordinary binding whose spelling is one of the non-primitive
built-in type names, in a scope where that type was visible, and from that
declaration to the end of the enclosing scope the name denotes only the new
binding. The built-in type is hidden: a use of the name in type position
inside that scope — an annotation, a type argument, a bound — no longer
resolves to a type and is an error (`E0101`). Likewise, shadowing an
intrinsic hides it: calls to the intrinsic inside that scope no longer
reach the intrinsic. Shadowing is what makes `let print = 0` legal and
`print("x")` afterwards an error — which is why both groups SHOULD be left
alone.

(The **Naming Conventions** table is unchanged: its Examples column gains
the lowercase spellings wherever a type example is shown —
`Map<string, int>`, `List<Image>` — through the rename defined in
[`../tasks.md`](../tasks.md).)
