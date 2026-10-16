# AST Schema — Delta: Header and Path Nodes

> **Delta** — replaces the **Program** section, the `DeclarationNode` union,
> the **Imports and re-exports** section, and the `VariantPathExpr` node and
> its union entry in `ast.md`.

## Program

```typescript
interface Program extends BaseNode {
  type: 'Program';
  header: HeaderItemNode[];
  body: DeclarationNode[];
}

type HeaderItemNode =
  | ModuleDeclaration
  | UseDeclaration
  | PubUseDeclaration;
```

This mirrors `Program = { HeaderItem } { Declaration } [ Entry ]`: the three
header forms first, then declarations. The grammar's `Entry` is not a
separate node — the root module's `main` is an ordinary `FnDeclaration` in
`body`, and the entry-point signature rule is a checking rule
([`modules/source-files.md`](modules/source-files.md)).

`DeclarationNode` loses `PubUseDeclaration`, which moved into
`HeaderItemNode`; declarations gain nothing — both `mod` forms are header
items, so `ModuleDeclaration` appears only there:

```typescript
type DeclarationNode =
  | FnDeclaration
  | StructDeclaration
  | EnumDeclaration
  | TraitDeclaration
  | ImplDeclaration
  | TypeAlias
  | LetDeclaration
  | ConstDeclaration
  | TestDeclaration;
```

`ModuleDeclaration` with `body: null` is the file-backed form (`mod form;`);
with a body it is the inline form. A checker recurses into
`ModuleDeclaration.header` / `ModuleDeclaration.body` exactly as it does
into `Program.header` / `Program.body` — an inline module is a module like
any other, one level down.

## Module and use nodes

```typescript
interface ModuleDeclaration extends BaseNode {
  type: 'ModuleDeclaration';
  isPub: boolean;
  name: IdentifierNode;
  header: HeaderItemNode[];
  body: DeclarationNode[] | null;
}

interface UseDeclaration extends BaseNode {
  type: 'UseDeclaration';
  isTypeOnly: boolean;
  target: UseTargetNode;
}

interface PubUseDeclaration extends BaseNode {
  type: 'PubUseDeclaration';
  target: UseTargetNode | null;   // null: the local form `pub use { a, b }`
  localNames: IdentifierNode[];   // empty unless the local form
}

type UseTargetNode = UsePathNode | UseGroupNode | UseGlobNode;

interface UsePathNode extends BaseNode {
  type: 'UsePath';
  path: PathNode;
  alias: IdentifierNode | null;   // the `as` name: a rename of the binding
}

interface UseGroupNode extends BaseNode {
  type: 'UseGroup';
  prefix: PathNode;
  entries: UseEntryNode[];
}

interface UseGlobNode extends BaseNode {
  type: 'UseGlob';
  prefix: PathNode;
}

interface UseEntryNode extends BaseNode {
  type: 'UseEntry';
  name: IdentifierNode;
  alias: IdentifierNode | null;
}

interface PathNode extends BaseNode {
  type: 'Path';
  head: PackageNameNode | IdentifierNode | 'self' | 'super';
  segments: IdentifierNode[];
}

interface PackageNameNode extends BaseNode {
  type: 'PackageName';
  identifiers: IdentifierNode[];  // one per hyphen-separated piece
}
```

The old import nodes — `TypeImportDeclaration`,
`NamedImportDeclaration`, `NamespaceImportDeclaration`,
`SideEffectImportDeclaration` — and the `source: string` field they shared
are gone: strings no longer name modules, `type` marks the whole
`UseDeclaration` rather than a list, and the namespace case is a `UsePath`
with an alias or a `UseGlob`. `UsePath.path` is a full `Path`, so
`use form::validate` and `use lsp-server as ls` share one node;
`PubUseDeclaration.target` distinguishes the path form from the local form,
whose entries are plain identifiers with no `as` — the export name is the
binding. Every `header` position (program and inline module) uses the same
`HeaderItemNode` union.

## Expressions — paths

```typescript
interface PathExpr extends BaseNode {
  type: 'PathExpr';
  path: PathNode;
  args: ArgNode[] | null;
}
```

`PathExpr` replaces `VariantPathExpr` in the `ExpressionNode` union and in
the grammar's `PrimaryExpression`: the same node now carries enum variants
(`Theme::Dark`, `Shape::Circle(2)`), module and package paths
(`form::validate(1, 2)`, `lsp-server::run()`), and associated names
(`Math::max(a, b)`, `Area::area(r)`). `PathNode.segments` is the whole chain
after the head, so a variant path is a two-segment path with `args`.
Pattern nodes are unchanged: a match pattern's variant spelling keeps its
own pattern node.
