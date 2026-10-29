# Grammar — Delta: Type Arguments After a Path

> **Delta** — widens the `CallTypeArgs` recognition rule in the Expressions
> section of `grammar.md` to admit a callee written as a `::`-path; the
> productions themselves are unchanged. This is the first construct that
> writes `<…>` after a path (`Array::filled<int, 3>(0)`).

Replace the opening of the `CallTypeArgs` recognition paragraph with:

`CallTypeArgs` is recognized only in call position, and only when the
expression before it ends in an identifier — a bare identifier, the final
segment of a `::`-path (`Array::filled`), or a member access — so that a
callee reached through a subscript, a call, or a closing bracket takes no
list.

A path callee reaches the list through `PostfixOperand`
(`PathExpr` stops at the path when the next token is `<`, and the list
follows as the operand's `CallTypeArgs`); the two disambiguation conditions
below apply unchanged, and a `<` after a path that fails either condition is
the relational operator, exactly as after a bare identifier.
