# AST — Delta: Const Generic Parameters

> **Delta** — replaces the `GenericParamNode` interface in `ast.md` and its
> explanatory sentence; the rest of the AST is unchanged.

```text
interface GenericParamNode extends BaseNode {
  type: 'GenericParam';
  kind: 'type' | 'const';
  name: IdentifierNode;
  bound: TypeNode | null;      // type parameters only
  constType: TypeNode | null;  // const parameters only
}
```

- `kind` distinguishes a type parameter from a const parameter. A type
  parameter has `bound` (or `null`) and `constType` is `null`; a const
  parameter has `constType` naming its value type (`int`) and `bound` is
  `null`. A written generic argument keeps its syntax node — an integer
  literal or a path — and the checker interprets it by the callee's parameter
  kind; there is no separate const-argument node.
