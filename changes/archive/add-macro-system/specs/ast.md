# AST — Delta: Macro Nodes

> **Delta** — replaces `AttributeNode`, adds `attributes` to `TypeAlias`,
> `LetDeclaration`, and `ConstDeclaration`, adds `MacroCallExpr`, `QuoteExpr`,
> and their supporting nodes to the Declarations and Expressions sections of
> `ast.md`, and notes that a `#[macro]` declaration is an ordinary
> `FnDeclaration`. The rest of the AST is unchanged.

## Declarations

Replace `AttributeNode` with:

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: AttributeNameNode;
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

`AttributeNameNode` is the node introduced by `change-attribute-spelling`;
`name.name` is the lowercase spelling. `args` is `null` for a bare attribute
(`#[route]`) and an array when a parenthesized group is written; the array is
empty for `#[route()]`. An `AttributeArgNode` with a non-null `name` is the
`name = value` form, which `#[macro]` uses
(`#[macro(kind = "derive", target = Serialize)]`); a positional argument has
`name: null` (`#[derive(Debug)]`, `#[route("/v1/users")]`). For a built-in
attribute, the registry entry decides which shape is valid; a user attribute
macro receives the values and validates them itself.

Replace `TypeAlias`, `LetDeclaration`, and `ConstDeclaration` with the
attribute-carrying forms:

```typescript
interface TypeAlias extends BaseNode {
  type: 'TypeAlias';
  attributes: AttributeNode[];
  isPub: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  initType: TypeNode;
}

interface LetDeclaration extends BaseNode {
  type: 'LetDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  isMutable: boolean;
  isShared: boolean;
  target: BindingTargetNode;
  varType: TypeNode | null;
  init: ExpressionNode;
}

interface ConstDeclaration extends BaseNode {
  type: 'ConstDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  name: IdentifierNode;
  varType: TypeNode | null;
  value: ExpressionNode;
}
```

Replace the note that lists these nodes as having no `attributes` field with:

> `TestDeclaration`, `PubUseDeclaration`, `ExternBlockDeclaration`, and
> `ExternFunctionDeclaration` carry no `attributes` field. The eight attribute
> hosts — `FnDeclaration`, `StructDeclaration`, `EnumDeclaration`,
> `TraitDeclaration`, `ImplDeclaration`, `TypeAlias`, `LetDeclaration`, and
> `ConstDeclaration` — each carry `attributes`.

A `#[macro]` definition adds no declaration node: it is an ordinary
`FnDeclaration` whose `attributes` contains the `AttributeNode` named
`macro`. The same is true of `#[derive]` and `#[test]` on their existing
nodes.

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
