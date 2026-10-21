---
title: AST Schema
description: Abstract syntax tree node definitions and data structures.
---

## Overview

This document defines the abstract syntax tree (AST) representation for Xulo programs. The schema is specified in TypeScript-style notation for clarity, but is language-agnostic.

Every construct named by [`grammar.md`](grammar.md) has a node here, and a node's shape mirrors the production it comes from: a field is `null` or an empty array exactly where the production makes its piece optional. Where the grammar names a construct but the AST has no node for it, the section says why.

## Node Base

All AST nodes share a common base:

```typescript
interface BaseNode {
  type: string;
  span: { start: number; end: number };
}
```

`type` is the discriminant: two nodes are the same kind of node exactly when their `type` strings are equal.

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

`ModuleDeclaration` with `body: null` is the file-backed form (`mod form;`);
with a body it is the inline form. A checker recurses into
`ModuleDeclaration.header` / `ModuleDeclaration.body` exactly as it does
into `Program.header` / `Program.body` — an inline module is a module like
any other, one level down.

## Declarations

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: IdentifierNode;
  args: IdentifierNode[] | null;
}

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

`abi` carries the grammar's `AbiTag` as a bare `StringLiteral`; an extern
parameter's `mode` and `default` are always `null`. `ExternItemNode.localName`
is the `as` target, `null` when an item binds its declared name;
`ExternFunctionDeclaration` has no `isPub` because the `pub` is mandatory —
its absence is a syntax failure, so no boolean distinguishes the cases.

```typescript
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

interface TraitMethodNode extends BaseNode {
  type: 'TraitMethod';
  isAsync: boolean;
  name: IdentifierNode;
  params: ParameterNode[];
  returnType: TypeNode;
}

interface ImplDeclaration extends BaseNode {
  type: 'ImplDeclaration';
  attributes: AttributeNode[];
  genericParams: GenericParamNode[];
  traitName: IdentifierNode | null;
  targetType: TypeNode;
  methods: FnDeclaration[];
}

interface TypeAlias extends BaseNode {
  type: 'TypeAlias';
  isPub: boolean;
  name: IdentifierNode;
  genericParams: GenericParamNode[];
  initType: TypeNode;
}

interface LetDeclaration extends BaseNode {
  type: 'LetDeclaration';
  isPub: boolean;
  isMutable: boolean;
  isShared: boolean;
  target: BindingTargetNode;
  varType: TypeNode | null;
  init: ExpressionNode;
}

interface ConstDeclaration extends BaseNode {
  type: 'ConstDeclaration';
  isPub: boolean;
  name: IdentifierNode;
  varType: TypeNode | null;
  value: ExpressionNode;
}

interface TestDeclaration extends BaseNode {
  type: 'TestDeclaration';
  name: IdentifierNode;
  title: StringLiteral | null;
  isAsync: boolean;
  body: BlockNode;
}
```

`title` is `null` exactly when the group is absent or empty — it is a
dedicated field, not an `attributes` entry, and the note that
`TestDeclaration` carries no `attributes` field is unchanged. The title is
metadata for reporting: it is not a name, and `TestDeclaration.name` keeps
every rule it has.

`TypeAlias`, `LetDeclaration`, `ConstDeclaration`, `TestDeclaration`, and
`PubUseDeclaration` carry no `attributes` field: the grammar attaches the
list to the five item forms only ([`grammar.md`](grammar.md)).

- Visibility is written `pub` on every declaration that has it. There is no `export` keyword, so there is no `isExport` field and no export node ([`grammar.md`](grammar.md), [`modules/visibility.md`](modules/visibility.md)).
- `impl` is never `pub` — it is never part of an interface — so `ImplDeclaration` carries no visibility flag at all.
- `TestDeclaration.name` is the test's identity for tools, not a name in the value namespace, so any reference to it is `E0107`, and it carries no `isPub` flag: a test is never part of an interface ([`testing.md`](testing.md)).
- `FnDeclaration.where` is the `where` clause as the list of items it holds, `null` when the clause is absent; only `FnDecl` takes one.
- `LetDeclaration.isMutable` is `true` for `let mut x = e` and for `let x := e`, which means exactly the same thing. `isShared` is `true` only for the `shared` marker written before the initializer; it is never `true` on a `:=` binding or on a destructuring binding. `target` is a name or a destructuring target, and a destructuring target carries no type annotation, so `varType` is `null` for it.

```typescript
type BindingTargetNode = IdentifierNode | DestructuringTargetNode;

interface DestructuringTargetNode extends BaseNode {
  type: 'DestructuringTarget';
  kind: 'struct' | 'tuple';
  names: IdentifierNode[];
}
```

### Component declarations

`ComponentDecl = StateDecl | StoreDecl | EffectDecl | EnvironmentDecl` is a closed set of four `@` forms, not a general attribute syntax: `grammar.md` admits no other `@` declaration, so the AST has four dedicated nodes rather than an `Attribute` wrapper carrying a name and arguments. General declaration attributes are the `#` forms — the `AttributeNode` above — and they appear on no component declaration; neither marker enters the other's territory ([`types/traits.md`](types/traits.md)).

```typescript
type ComponentDeclaration =
  | StateDeclaration
  | StoreDeclaration
  | EffectDeclaration
  | EnvironmentDeclaration;

interface StateDeclaration extends BaseNode {
  type: 'StateDeclaration';
  name: IdentifierNode;
  varType: TypeNode | null;
  init: ExpressionNode;
}

interface StoreDeclaration extends BaseNode {
  type: 'StoreDeclaration';
  names: IdentifierNode[];
  init: ExpressionNode;
}

interface EffectDeclaration extends BaseNode {
  type: 'EffectDeclaration';
  value: ExpressionNode;
  dependencies: ExpressionNode | null;
}

interface EnvironmentDeclaration extends BaseNode {
  type: 'EnvironmentDeclaration';
  name: IdentifierNode;
  varType: TypeNode;
}
```

`EffectDeclaration.dependencies` is the optional expression written after the `,` — a list literal when the declaration is well-formed, and anything else when it is not, which is the shape `E0708` diagnoses. `EnvironmentDeclaration.varType` is required: the production takes no optional annotation, and a declaration written without one is `E0710`.

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

## Parameters and Fields

```typescript
type ParameterNode = ValueParameter | Receiver;

interface ValueParameter extends BaseNode {
  type: 'Parameter';
  name: IdentifierNode;
  mode: 'mut' | 'shared' | null;
  paramType: TypeNode;
  default: ExpressionNode | null;
}

interface Receiver extends BaseNode {
  type: 'Receiver';
  isMutable: boolean;
}

interface ClosureParamNode extends BaseNode {
  type: 'ClosureParam';
  name: IdentifierNode;
  paramType: TypeNode | null;
}

interface GenericParamNode extends BaseNode {
  type: 'GenericParam';
  name: IdentifierNode;
  bound: TypeNode | null;
}

interface WhereItemNode extends BaseNode {
  type: 'WhereItem';
  name: IdentifierNode;
  bounds: TypeNode[];
}

interface FieldNode extends BaseNode {
  type: 'Field';
  isPub: boolean;
  name: IdentifierNode;
  fieldType: TypeNode;
}

interface EnumVariantNode extends BaseNode {
  type: 'EnumVariant';
  name: IdentifierNode;
  payload: EnumPayloadSlotNode[];
}

interface EnumPayloadSlotNode extends BaseNode {
  type: 'EnumPayloadSlot';
  name: IdentifierNode | null;
  slotType: TypeNode;
}
```

- A declared function's parameter is always annotated — `Parameter = Receiver | Identifier ":" [ "mut" | "shared" ] Type [ "=" Expression ]` — so `ValueParameter.paramType` is never `null`, and a parameter is optional exactly when it has a `default`. `mode` is `null` unless the type wrote `mut` or `shared`; the mode is part of the parameter, not of its type.
- `Receiver` is the `self` / `mut self` alternative of `Parameter`; a parameter list that begins with one carries `Receiver` rather than a name and a type.
- A closure's parameter is written `ClosureParam = Identifier [ ":" Type ]`, so its annotation may be omitted and `ClosureParamNode.paramType` is `null` in that case; the two parameter nodes are separate because the grammar gives the two forms separately.
- `GenericParamNode.bound` is the `TraitBound` written after `:`, `null` when the parameter is unbounded. A bound joined with `+` is a list of types, as in `WhereItemNode.bounds`. A `TraitDeclaration`'s `genericParams` is always empty: a trait may not declare type parameters, and an attempt is diagnosed after parsing (`E0810`; [`types/traits.md`](types/traits.md)).
- A variant with no payload has an empty `payload`. Each slot's `name` is `null`
  for a positional payload and non-`null` for a named one, and a variant never
  mixes the two forms: either every slot is named or none is
  (`VariantPayload = "(" ( TypeList | NameTypeList ) ")"` in
  [`grammar.md`](grammar.md), [`types/enums.md`](types/enums.md)).

## Types

```typescript
type TypeNode =
  | IdentifierType
  | GenericType
  | OptionalType
  | UnionType
  | IntersectionType
  | TupleType
  | LiteralType
  | FnType;

interface IdentifierType extends BaseNode {
  type: 'IdentifierType';
  name: string;
}

interface GenericType extends BaseNode {
  type: 'GenericType';
  base: TypeNode;
  args: TypeNode[];
}

interface OptionalType extends BaseNode {
  type: 'OptionalType';
  inner: TypeNode;
}

interface UnionType extends BaseNode {
  type: 'UnionType';
  left: TypeNode;
  right: TypeNode;
}

interface IntersectionType extends BaseNode {
  type: 'IntersectionType';
  left: TypeNode;
  right: TypeNode;
}

interface TupleType extends BaseNode {
  type: 'TupleType';
  elements: TypeNode[];
}

interface LiteralType extends BaseNode {
  type: 'LiteralType';
  value: NumberLiteral | StringLiteral | BooleanLiteral;
}

interface FnType extends BaseNode {
  type: 'FnType';
  isAsync: boolean;
  params: FnTypeParamNode[];
  returnType: TypeNode | null;
}

interface FnTypeParamNode extends BaseNode {
  type: 'FnTypeParam';
  name: IdentifierNode | null;
  mode: 'mut' | 'shared' | null;
  paramType: TypeNode;
}
```

`UnionType` and `IntersectionType` are binary because the productions are
left-nesting lists: `A | B | C` is `UnionType(UnionType(A, B), C)`. A tuple
type needs at least two elements (`TupleType`), and `LiteralType` carries the
literal it names — a string, integer, float, or boolean used as a type.

A function-type parameter is written `FnTypeParam = [ Identifier ":" ] [
"mut" | "shared" ] Type`. Both trailing fields are carried: `name` is `null`
unless the type wrote one, and `mode` is `null` unless the type wrote `mut` or
`shared`. The name is documentation — two function types that differ only in
parameter names are the same type — but the mode is part of the type, so
`fn(T): R`, `fn(mut T): R`, and `fn(shared T): R` are three distinct types
([`types/function-types.md`](types/function-types.md)).

## Expressions

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

interface NumberLiteral extends BaseNode {
  type: 'NumberLiteral';
  kind: 'integer' | 'float';
  raw: string;
}

interface StringLiteral extends BaseNode {
  type: 'StringLiteral';
  raw: string;
  cooked: string;
}

interface BooleanLiteral extends BaseNode {
  type: 'BooleanLiteral';
  value: boolean;
}

interface NullLiteral extends BaseNode {
  type: 'NullLiteral';
}

interface BinaryExpr extends BaseNode {
  type: 'BinaryExpr';
  op: string;
  left: ExpressionNode;
  right: ExpressionNode;
}

interface RangeExpr extends BaseNode {
  type: 'RangeExpr';
  op: '..' | '..<';
  start: ExpressionNode;
  end: ExpressionNode;
}

interface IsExpr extends BaseNode {
  type: 'IsExpr';
  operand: ExpressionNode;
  target: TypeNode;
  negated: boolean;
}

interface UnaryExpr extends BaseNode {
  type: 'UnaryExpr';
  op: '!' | '-' | '~' | 'move' | 'copy';
  argument: ExpressionNode;
}

interface AwaitExpr extends BaseNode {
  type: 'AwaitExpr';
  argument: ExpressionNode;
}

interface CallExpr extends BaseNode {
  type: 'CallExpr';
  callee: ExpressionNode;
  args: ArgNode[];
}

interface MemberExpr extends BaseNode {
  type: 'MemberExpr';
  object: ExpressionNode;
  property: IdentifierNode | NumberLiteral;
  optional: boolean;
}

interface IndexExpr extends BaseNode {
  type: 'IndexExpr';
  object: ExpressionNode;
  index: ExpressionNode;
}

interface PropagateExpr extends BaseNode {
  type: 'PropagateExpr';
  argument: ExpressionNode;
}

interface TernaryExpr extends BaseNode {
  type: 'TernaryExpr';
  test: ExpressionNode;
  consequent: ExpressionNode;
  alternate: ExpressionNode;
}

interface IfExpr extends BaseNode {
  type: 'IfExpr';
  condition: ExpressionNode;
  consequent: BlockNode;
  alternate: IfExpr | BlockNode | null;
}

interface MatchExpr extends BaseNode {
  type: 'MatchExpr';
  subject: ExpressionNode;
  arms: MatchArmNode[];
}

interface MatchArmNode extends BaseNode {
  type: 'MatchArm';
  pattern: PatternNode;
  guard: ExpressionNode | null;
  body: ExpressionNode | BlockNode;
}

interface ClosureExpr extends BaseNode {
  type: 'ClosureExpr';
  isAsync: boolean;
  params: ClosureParamNode[];
  returnType: TypeNode | null;
  body: ExpressionNode | BlockNode;
}

interface PanicExpr extends BaseNode {
  type: 'PanicExpr';
  argument: ExpressionNode;
}

interface ListExpr extends BaseNode {
  type: 'ListExpr';
  elements: (ExpressionNode | SpreadNode)[];
}

type MapExpr = TypedMapExpr | BraceMapExpr;

interface TypedMapExpr extends BaseNode {
  type: 'TypedMapExpr';
  keyType: TypeNode;
  valueType: TypeNode;
  entries: MapEntryNode[];
}

interface BraceMapExpr extends BaseNode {
  type: 'BraceMapExpr';
  entries: (MapEntryNode | MapFieldNode | SpreadNode)[];
}

interface MapEntryNode extends BaseNode {
  type: 'MapEntry';
  key: ExpressionNode;
  value: ExpressionNode;
}

interface MapFieldNode extends BaseNode {
  type: 'MapField';
  name: IdentifierNode;
  value: ExpressionNode;
}

interface TupleExpr extends BaseNode {
  type: 'TupleExpr';
  elements: ExpressionNode[];
}

interface PathExpr extends BaseNode {
  type: 'PathExpr';
  path: PathNode;
  args: ArgNode[] | null;
}

`PathExpr` replaces `VariantPathExpr` in the `ExpressionNode` union and in
the grammar's `PrimaryExpression`: the same node now carries enum variants
(`Theme::Dark`, `Shape::Circle(2)`), module and package paths
(`form::validate(1, 2)`, `lsp-server::run()`), and associated names
(`Math::max(a, b)`, `Area::area(r)`). `PathNode.segments` is the whole chain
after the head, so a variant path is a two-segment path with `args`.
Pattern nodes are unchanged: a match pattern's variant spelling keeps its
own pattern node.

type SpawnExpr = BlockSpawnExpr | ServiceSpawnExpr;

interface BlockSpawnExpr extends BaseNode {
  type: 'BlockSpawnExpr';
  mode: 'default' | 'thread' | 'process' | 'on';
  on: ExpressionNode | null;
  body: BlockNode;
}

interface ServiceSpawnExpr extends BaseNode {
  type: 'ServiceSpawnExpr';
  service: TypeNode;
  fields: FieldInitNode[];
}

interface LockExpr extends BaseNode {
  type: 'LockExpr';
  target: IdentifierNode;
  body: BlockNode;
}

interface ComponentExpr extends BaseNode {
  type: 'ComponentExpr';
  name: IdentifierNode;
  args: ArgNode[];
  children: ComponentBlockNode | null;
}

interface TemplateLiteralNode extends BaseNode {
  type: 'TemplateLiteral';
  quasis: TemplateElementNode[];
  expressions: ExpressionNode[];
}

interface TemplateElementNode extends BaseNode {
  type: 'TemplateElement';
  value: {
    raw: string;
    cooked: string;
  };
  tail: boolean;
}
```

- `BinaryExpr.op` is the operator as written, and the node covers every level of the precedence table that takes two expressions: arithmetic, the word operators `and` / `or`, bitwise and shift, `==` / `!=`, the relational operators, and the nullish `??`. `is` is not one of them — it takes a type on the right, which is why it is `IsExpr` — and neither is a range, which is `RangeExpr` with `op` naming which end is open.
- `UnaryExpr.op` covers the prefix operators that are not `await`: `await` has its own node because it is the only one whose operand must be a `Task`.
- `MemberExpr.optional` distinguishes `.` from `?.`. The property is a name or an integer literal, since the production admits both.
- `MapExpr` is a union because `Map<K, V>{ … }` and a brace map take different pieces: a typed map has type arguments and entries only, while a brace map has fields, entries written as `key: value`, and spreads.
- `SpawnExpr` is a union because `SpawnExpression` has two unrelated forms: `spawn [mode] async Block` runs a block, and `spawn .process Type { … }` starts a service. `BlockSpawnExpr.mode` is `default` when the target is written bare, and `on` is the only mode that carries an expression.
- `LockExpr.target` is a bare identifier, matching `LockExpression = "lock" Identifier Block`.
- A component invocation that carries a children block is a `ComponentExpr`; one written with an argument list and no block parses as an ordinary call through `PostfixOperand` and is therefore a `CallExpr`. The two read the same ([`grammar.md`](grammar.md), [`components/view-syntax.md`](components/view-syntax.md)).
- The union has no node for `AssignmentExpression`: an assignment is written only through the `Assignment` statement, and `ExpressionStatement = TernaryExpression` stops below it ([`grammar.md`](grammar.md)).

## Patterns

```typescript
type PatternNode =
  | WildcardPattern
  | IdentifierPattern
  | LiteralPattern
  | RangePattern
  | VariantPattern
  | StructPattern
  | TypePattern
  | OrPattern;

interface WildcardPattern extends BaseNode {
  type: 'WildcardPattern';
}

interface IdentifierPattern extends BaseNode {
  type: 'IdentifierPattern';
  name: IdentifierNode;
}

interface LiteralPattern extends BaseNode {
  type: 'LiteralPattern';
  value: NumberLiteral | StringLiteral | BooleanLiteral | NullLiteral;
}

interface RangePattern extends BaseNode {
  type: 'RangePattern';
  start: NumberLiteral;
  end: NumberLiteral;
  endInclusive: boolean;
}

interface VariantPattern extends BaseNode {
  type: 'VariantPattern';
  enumName: IdentifierNode;
  variantName: IdentifierNode;
  bindings: PatternNode[] | null;
}

interface StructPattern extends BaseNode {
  type: 'StructPattern';
  name: IdentifierNode;
  bindings: PatternNode[] | null;
}

interface TypePattern extends BaseNode {
  type: 'TypePattern';
  designators: PatternDesignatorNode[];
  binder: IdentifierNode | WildcardPattern;
}

interface OrPattern extends BaseNode {
  type: 'OrPattern';
  alternatives: PatternNode[];
}

interface PatternDesignatorNode extends BaseNode {
  type: 'PatternDesignator';
  name: IdentifierNode;
  optional: boolean;
}
```

`Pattern = PatternAlt { "|" PatternAlt }` has eight `PatternAlt` alternatives — the wildcard is one of them even though it lexes as an identifier: `_` is `WildcardPattern`, any other bare identifier is `IdentifierPattern`, which binds — and a disjunction of two or more alternatives parses as one `OrPattern` whose `alternatives` is a flat list, so `a | b | c` is three alternatives and never nested pairs. A parenthesized alternative parses as the pattern inside it and adds no node of its own. `MatchArmNode.guard` is the arm's guard expression, `null` when the arm has none. `RangePattern.endInclusive` records `...` versus `..<`, and both operands are numeric literals, matching the production. `bindings` is `null` when the pattern writes no parentheses and an array otherwise.

`TypePattern.binder` is the name or `_` that follows the designators, and `designators` is the possibly unioned list of type names, each with the `optional` flag its `?` carries.

## Statements

```typescript
type StatementNode =
  | FnDeclaration
  | LetDeclaration
  | ConstDeclaration
  | ComponentDeclaration
  | ReturnStatement
  | BreakStatement
  | ContinueStatement
  | DeferStatement
  | AssignmentNode
  | ExpressionStatement
  | ForStatement
  | WhileStatement;

interface ReturnStatement extends BaseNode {
  type: 'ReturnStatement';
  argument: ExpressionNode | null;
}

interface BreakStatement extends BaseNode {
  type: 'BreakStatement';
}

interface ContinueStatement extends BaseNode {
  type: 'ContinueStatement';
}

interface DeferStatement extends BaseNode {
  type: 'DeferStatement';
  operand: ExpressionNode | BlockNode;
}

interface AssignmentNode extends BaseNode {
  type: 'Assignment';
  target: PlaceNode;
  value: ExpressionNode;
}

interface ExpressionStatement extends BaseNode {
  type: 'ExpressionStatement';
  expression: ExpressionNode;
}

interface ForStatement extends BaseNode {
  type: 'ForStatement';
  variable: IdentifierNode;
  iterable: ExpressionNode;
  body: BlockNode;
}

interface WhileStatement extends BaseNode {
  type: 'WhileStatement';
  condition: ExpressionNode;
  body: BlockNode;
}
```

`Statement = ( FnDecl | LetDecl | ConstDecl | ComponentDecl | ReturnStmt | BreakStmt | ContinueStmt | DeferStmt | Assignment | ExpressionStatement | ForStmt | WhileStmt ) StatementEnd`. A `Block` is not one of them — a block is the body of a construct, never a statement — and the trailing `;` is an optional terminator the AST does not store, since `span` already covers it. `AssignmentNode.target` is a `PlaceNode`, not a general expression, which is what makes an assignment a statement rather than an expression. `DeferStatement.operand` is an `ExpressionNode` or a `BlockNode`, recorded as written; the operand's value is discarded when it runs.

## Common Nodes

```typescript
interface IdentifierNode extends BaseNode {
  type: 'Identifier';
  value: string;
}

interface BlockNode extends BaseNode {
  type: 'Block';
  body: StatementNode[];
}

interface ComponentBlockNode extends BaseNode {
  type: 'ComponentBlock';
  children: ChildItemNode[];
}

type ChildItemNode = ExpressionNode | ForStatement;

interface ArgNode extends BaseNode {
  type: 'Arg';
  name: IdentifierNode | null;
  value: ExpressionNode | BindingExprNode;
}

interface BindingExprNode extends BaseNode {
  type: 'BindingExpr';
  place: PlaceNode;
}

type PlaceNode = IdentifierNode | MemberPlaceNode;

interface MemberPlaceNode extends BaseNode {
  type: 'MemberPlace';
  object: PlaceNode;
  property: IdentifierNode | NumberLiteral;
}

interface SpreadNode extends BaseNode {
  type: 'Spread';
  argument: ExpressionNode;
}

interface FieldInitNode extends BaseNode {
  type: 'FieldInit';
  name: IdentifierNode;
  value: ExpressionNode;
}
```

`BlockNode` holds statements and is the body of a function, an `if`, a `for`,
or a `while`. A component's children block is a different node:
`ComponentBlock = "{" { ChildItem } "}"` with `ChildItem = Expression |
ForStmt`, so its `children` are child items and never statements, bindings, or
assignments ([`grammar.md`](grammar.md),
[`components/view-syntax.md`](components/view-syntax.md)).

`ArgNode.name` is the argument label, `null` when the argument is positional.
Its `value` is an expression, or — when the argument is written `$place` — a
`BindingExprNode`, which is not an expression: a `$` argument does not evaluate
to something a program may bind, store, index, or pass on, and its `place` is
the `Place` the grammar accepts there, so a dotted place such as `user.name`
is representable and is rejected by the checker as `E0712`
([`components/binding.md`](components/binding.md)).

`PlaceNode` mirrors `Place = Identifier | Place "." Identifier | Place "." IntegerLiteral`, so it is never a general expression; it is what an assignment target and a `$` place are made of.

## Serialization Format

AST nodes **SHALL** be serializable to JSON. The `span` field records zero-based character offsets `(start, end)` in the source file. A union is serialized as the value of its `type` field together with the fields that node declares; a `null` field means the production left the piece out, and an empty array means it repeated zero times.
