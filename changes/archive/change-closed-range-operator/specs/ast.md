# AST Schema — Delta: Closed Range Operator

> **Delta** — replaces the `RangeExpr` interface and two prose sentences in
> `ast.md`.

## Expressions

`RangeExpr` becomes:

```ts
interface RangeExpr extends BaseNode {
  type: 'RangeExpr';
  op: '..=' | '..<';
  start: ExpressionNode;
  end: ExpressionNode;
}
```

The `BinaryExpr` bullet becomes:

- `BinaryExpr.op` is the operator as written, and the node covers every level of the precedence table that takes two expressions: arithmetic, the word operators `and` / `or`, bitwise and shift, `==` / `!=`, the relational operators, and the nullish `??`. `is` is not one of them — it takes a type on the right, which is why it is `IsExpr` — and neither is a range, which is `RangeExpr` with `op` naming whether the upper bound is included: `..=` includes it, `..<` excludes it.

## Patterns

The sentence describing `RangePattern` becomes:

`RangePattern.endInclusive` records `..=` versus `..<`, and both operands are
numeric literals, matching the production.
