---
title: AST Schema
description: Abstract syntax tree node definitions and data structures.
---

## Overview

This document defines the abstract syntax tree (AST) representation for XULO programs. The schema is specified in TypeScript-style notation for clarity, but is language-agnostic.

## Node Base

All AST nodes share a common base:

```typescript
interface BaseNode {
  type: string;
  span: { start: number; end: number };
}
```

## Program

```typescript
interface Program extends BaseNode {
  type: 'Program';
  body: Declaration[];
}
```

## Declarations

```typescript
interface FnDeclaration extends BaseNode {
  type: 'FnDeclaration';
  isExport: boolean;
  isAsync: boolean;
  name: IdentifierNode;
  genericParams: IdentifierNode[];
  params: ParameterNode[];
  returnType: TypeNode | null;
  body: BlockNode;
}

interface StructDeclaration extends BaseNode {
  type: 'StructDeclaration';
  isExport: boolean;
  isPub: boolean;
  name: IdentifierNode;
  genericParams: IdentifierNode[];
  fields: FieldNode[];
}

interface EnumDeclaration extends BaseNode {
  type: 'EnumDeclaration';
  isExport: boolean;
  name: IdentifierNode;
  genericParams: IdentifierNode[];
  variants: EnumVariantNode[];
}

interface TraitDeclaration extends BaseNode {
  type: 'TraitDeclaration';
  isExport: boolean;
  name: IdentifierNode;
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
  isExport: boolean;
  genericParams: IdentifierNode[];
  traitName: IdentifierNode | null;
  targetType: TypeNode;
  methods: FnDeclaration[];
}

interface TypeAlias extends BaseNode {
  type: 'TypeAlias';
  isExport: boolean;
  name: IdentifierNode;
  genericParams: IdentifierNode[];
  initType: TypeNode;
}
```

## Parameters and Fields

```typescript
interface ParameterNode extends BaseNode {
  type: 'Parameter';
  name: IdentifierNode;
  paramType: TypeNode;
  isOptional: boolean;
  default: ExpressionNode | null;
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
  payload: TypeNode[];
}
```

## Types

```typescript
type TypeNode =
  | IdentifierType
  | GenericType
  | OptionalType
  | UnionType
  | IntersectionType
  | FnType
  | ObjectType;

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

interface FnType extends BaseNode {
  type: 'FnType';
  isAsync: boolean;
  params: TypeNode[];
  returnType: TypeNode | null;
}
```

## Expressions

```typescript
type ExpressionNode =
  | IdentifierNode
  | NumberLiteral
  | StringLiteral
  | BooleanLiteral
  | NullLiteral
  | BinaryExpr
  | UnaryExpr
  | CallExpr
  | MemberExpr
  | IndexExpr
  | TernaryExpr
  | IfExpr
  | MatchExpr
  | ClosureExpr
  | AwaitExpr
  | ListExpr
  | ObjectExpr
  | ComponentExpr
  | TemplateLiteralNode;

interface BinaryExpr extends BaseNode {
  type: 'BinaryExpr';
  op: string;
  left: ExpressionNode;
  right: ExpressionNode;
}

interface CallExpr extends BaseNode {
  type: 'CallExpr';
  callee: ExpressionNode;
  args: ExpressionNode[];
}

interface ComponentExpr extends BaseNode {
  type: 'ComponentExpr';
  name: IdentifierNode;
  args: ArgNode[];
  body: BlockNode | null;
}

interface MatchExpr extends BaseNode {
  type: 'MatchExpr';
  subject: ExpressionNode;
  arms: MatchArmNode[];
}

interface ClosureExpr extends BaseNode {
  type: 'ClosureExpr';
  isAsync: boolean;
  params: ParameterNode[];
  returnType: TypeNode | null;
  body: ExpressionNode | BlockNode;
}

interface AwaitExpr extends BaseNode {
  type: 'AwaitExpr';
  argument: ExpressionNode;
}

interface IfExpr extends BaseNode {
  type: 'IfExpr';
  condition: ExpressionNode;
  consequent: BlockNode;
  alternate: IfExpr | BlockNode | null;
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

## Statements

```typescript
type StatementNode =
  | VariableDecl
  | ReturnStatement
  | ThrowStatement
  | ExpressionStatement
  | ForStatement
  | WhileStatement
  | TryStatement;

interface VariableDecl extends BaseNode {
  type: 'VariableDecl';
  isMutable: boolean;
  name: IdentifierNode;
  varType: TypeNode | null;
  init: ExpressionNode;
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

interface TryStatement extends BaseNode {
  type: 'TryStatement';
  body: BlockNode;
  catches: CatchClause[];
  finally: BlockNode | null;
}

interface CatchClause extends BaseNode {
  type: 'CatchClause';
  variable: IdentifierNode;
  body: BlockNode;
}
```

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

interface NamedArgNode extends BaseNode {
  type: 'NamedArg';
  name: IdentifierNode;
  value: ExpressionNode;
}

interface ArgNode extends BaseNode {
  type: 'Arg';
  name: IdentifierNode | null;
  value: ExpressionNode;
}

interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: IdentifierNode;
  args: ArgNode[];
}

interface ExportDeclaration extends BaseNode {
  type: 'ExportDeclaration';
  items: ExportItemNode[];
  source: string | null;
}

interface ExportItemNode extends BaseNode {
  type: 'ExportItem';
  name: IdentifierNode;
  alias: IdentifierNode | null;
}

interface ImportDeclaration extends BaseNode {
  type: 'ImportDeclaration';
  items: ImportItemNode[];
  source: string;
}

interface ImportItemNode extends BaseNode {
  type: 'ImportItem';
  name: IdentifierNode;
  alias: IdentifierNode | null;
}

interface UseDeclaration extends BaseNode {
  type: 'UseDeclaration';
  path: IdentifierNode[];
}
```

## Serialization Format

AST nodes **SHALL** be serializable to JSON. The `span` field records zero-based character offsets `(start, end)` in the source file.
