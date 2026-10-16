# AST Schema — Delta: Declaration Attributes

> **Delta** — adds an `AttributeNode`, adds an `attributes` field to the five
> item declaration interfaces of `ast.md`, and revises the component-family
> note.

A declaration attribute is its own node; the argument list is present
(`[]`) exactly when the production writes no parenthesized list:

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: IdentifierNode;
  args: IdentifierNode[] | null;
}
```

The five item declaration interfaces gain an `attributes` field, ordered as
written; every other declaration node is unchanged:

```typescript
interface FnDeclaration extends BaseNode {
  type: 'FnDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  isAsync: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  where: WhereItemNode[] | null;
  params: ParameterNode[];
  returnType: TypeNode | null;
  body: BlockNode;
}

interface StructDeclaration extends BaseNode {
  type: 'StructDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  fields: FieldNode[];
}

interface EnumDeclaration extends BaseNode {
  type: 'EnumDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  variants: EnumVariantNode[];
}

interface TraitDeclaration extends BaseNode {
  type: 'TraitDeclaration';
  attributes: AttributeNode[];
  isPub: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  methods: TraitMethodNode[];
}

interface ImplDeclaration extends BaseNode {
  type: 'ImplDeclaration';
  attributes: AttributeNode[];
  genericParams: GenericParamNode[];
  traitName: IdentifierNode | null;
  targetType: TypeNode;
  methods: FnDeclaration[];
}
```

`TypeAlias`, `LetDeclaration`, `ConstDeclaration`, `TestDeclaration`, and
`PubUseDeclaration` carry no `attributes` field: the grammar attaches the
list to the five item forms only
([`grammar.md`](grammar.md)).

The component-family note becomes:

> `ComponentDecl = StateDecl | StoreDecl | EffectDecl | EnvironmentDecl` is a
> closed set of four `@` forms, not a general attribute syntax: `grammar.md`
> admits no other `@` declaration, so the AST has four dedicated nodes rather
> than an `Attribute` wrapper carrying a name and arguments. General
> declaration attributes are the `#` forms — the `AttributeNode` above — and
> they appear on no component declaration; neither marker enters the other's
> territory ([`types/traits.md`](types/traits.md)).
