# Intrinsic Functions — Delta: StringBuilder

> **Delta** — adds the `StringBuilder` row to the output rendering table and
> names `StringBuilder` among the operands without a string form in the
> Conversion section of `builtins/intrinsic-functions.md`; the rest is
> unchanged.

Add to the output rendering table, after the `string` row:

| `StringBuilder` | the current text of the buffer, exactly the `string` row — no surrounding quotes and no escaping |

Replace the “Any other operand…” sentence with:

Any other operand is a compile-time error (`E0212`) — a `List`, an `Array`, a
`Map`, a `Set`, a `StringBuilder` (materialize it with `.build()`), a `struct`
or `enum` with no `impl Display`, or `null`
([`../type-system/errors.md`](../type-system/errors.md)).
