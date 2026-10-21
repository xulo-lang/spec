# AST Schema — Delta: Extern Nodes

> **Delta** — edits the `DeclarationNode` union in the **Program** section
> and adds three node interfaces to the **Declarations** section of `ast.md`.

## The union

`DeclarationNode` gains two members, between `FnDeclaration` and
`StructDeclaration`:

```typescript
type DeclarationNode =
  | FnDeclaration
  | ExternBlockDeclaration
  | ExternFunctionDeclaration
  | StructDeclaration
  | EnumDeclaration
  | TraitDeclaration
  | ImplDeclaration
  | TypeAlias
  | LetDeclaration
  | ConstDeclaration
  | TestDeclaration;
```

The comment above the union — about `PubUseDeclaration` and the `mod` forms —
is unchanged: extern declarations are ordinary declarations of a module body.

## The nodes

Three interfaces join the **Declarations** section, immediately after
`FnDeclaration`:

```typescript
interface ExternBlockDeclaration extends BaseNode {
  type: 'ExternBlockDeclaration';
  isPub: boolean;
  abi: StringLiteral;
  items: ExternItemNode[];
}

interface ExternItemNode extends BaseNode {
  type: 'ExternItem';
  name: IdentifierNode;
  params: ParameterNode[];
  returnType: TypeNode | null;
  localName: IdentifierNode | null;
}

interface ExternFunctionDeclaration extends BaseNode {
  type: 'ExternFunctionDeclaration';
  abi: StringLiteral;
  name: IdentifierNode;
  params: ParameterNode[];
  returnType: TypeNode | null;
  body: BlockNode;
}
```

Notes, following the section's shape-mirrors-the-production rule:

- `abi` carries the `StringLiteral` of the grammar's `AbiTag` directly; a
  dedicated `AbiTagNode` would wrap exactly one field with no grammar of its
  own, so the wrapper is elided (`TestDeclaration.title` sets the precedent).
- `params` are `ParameterNode`s whose `mode` and `default` are always `null`
  — the field is `null` exactly where the production makes its piece absent,
  which for `ExternParam` is always ([`grammar.md`](grammar.md)).
- `ExternItemNode` has no `body`, no `attributes`, and no `isPub`: none exist
  in the production, and an item's export follows its block's `isPub`.
  `localName` is the `as` target, `null` when absent — an imported name bound
  to itself, as `use path::x` binds `x` to `x`.
- `ExternFunctionDeclaration` has no `isPub` (the `pub` is mandatory and its
  absence is a syntax failure, so no boolean distinguishes the cases), no
  `attributes`, no `isAsync`, no `genericParams`, and no `where` — the
  positions `FnDeclaration` carries and the extern form forbids.
