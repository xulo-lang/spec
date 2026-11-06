# AST — Delta: Macro Nodes

> **Delta** — replaces `AttributeNode`, adds `MacroCallExpr`, `QuoteExpr`, and
> their supporting nodes to the Declarations and Expressions sections of
> `ast.md`, and notes that a `#Macro` declaration is an ordinary
> `FnDeclaration`. The rest of the AST is unchanged.

## Declarations

Replace `AttributeNode` with:

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: IdentifierNode;
  args: AttributeArgNode[] | null;
}

interface AttributeArgNode extends BaseNode {
  type: 'AttributeArg';
  name: IdentifierNode | null;   // null: positional; otherwise `name = value`
  value: AttributeValueNode;
}

type AttributeValueNode =
  | IdentifierNode
  | StringLiteral
  | NumberLiteral
  | BooleanLiteral;
```

`args` is `null` for a bare attribute (`#Test`) and an array when a
parenthesized group is written; the array is empty for `#Test()`. An
`AttributeArgNode` with a non-null `name` is the `name = value` form, which
`#Macro` uses (`#Macro(kind = "derive", target = Serialize)`); a positional
argument has `name: null` (`#Derive(Debug)`, `#route("/v1/users")`). The registry
entry decides which shape is valid.

A `#Macro` definition adds no declaration node: it is an ordinary
`FnDeclaration` whose `attributes` contains the `AttributeNode` named
`Macro`. The same is true of `#Derive` and `#Test` on their existing nodes.

## Expressions

Add `MacroCallExpr` and `QuoteExpr` to the `ExpressionNode` union:

```typescript
type ExpressionNode =
  | IdentifierNode
  | NumberLiteral
  | StringLiteral
  | BooleanLiteral
  | NullLiteral
  | TemplateLiteralNode
  | BinaryExpr
  | RangeExpr
  | IsExpr
  | UnaryExpr
  | AwaitExpr
  | CallExpr
  | MacroCallExpr          // new
  | QuoteExpr              // new
  | MemberExpr
  | IndexExpr
  | PropagateExpr
  | TernaryExpr
  | IfExpr
  | MatchExpr
  | ClosureExpr
  | PanicExpr
  | ListExpr
  | MapExpr
  | TupleExpr
  | PathExpr
  | SpawnExpr
  | LockExpr
  | ComponentExpr;
```

The two new nodes:

```typescript
interface MacroCallExpr extends BaseNode {
  type: 'MacroCallExpr';
  path: PathNode;
  args: ExpressionNode[];
}

interface QuoteExpr extends BaseNode {
  type: 'QuoteExpr';
  pieces: QuotePieceNode[];
}

type QuotePieceNode = QuoteTokenNode | QuoteHoleNode;

interface QuoteTokenNode extends BaseNode {
  type: 'QuoteToken';
  text: string;
}

interface QuoteHoleNode extends BaseNode {
  type: 'QuoteHole';
  expression: ExpressionNode;
}
```

`MacroCallExpr.args` is always an array — empty for `sql!()` — because the
grammar has no bare `Path "!"` form without parentheses. The path is a
`PathNode`, so `sql!` and `m::sql!` are the same node shape. `QuoteExpr` keeps
the quote body unparsed: `QuoteTokenNode.text` is the raw balanced token text
between holes, and `QuoteHoleNode.expression` is the `${…}` operand. The
compiler parses the substituted body at expansion time, so the AST does not
need a hole-bearing variant of every declaration and expression node.

`QuoteExpr` is the only `QuoteCall` shape; there is no user macro with a
brace-delimited body. `MacroCallExpr` is a primary expression, not a
`CallExpr`: its callee names a macro, not a value, and it has no type
arguments or named arguments.
