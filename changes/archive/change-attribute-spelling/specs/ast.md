# AST Schema — Delta: Lowercase Attribute Brackets

> **Delta** — replaces `AttributeNode`, adds `AttributeNameNode`, adds a
> source note after them, removes `TestDeclaration.title` and its paragraph,
> and replaces the declaration-attribute sentence in the component note of
> `ast.md`.

Replace `AttributeNode` with:

```typescript
interface AttributeNameNode extends BaseNode {
  type: 'AttributeName';
  name: string;
}

interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: AttributeNameNode;
  args: IdentifierNode[] | null;
}
```

An attribute's `span` covers the leading `#` through the closing `]`.
`AttributeNameNode.name` is the spelling written between the brackets:
`derive`, `test`, `macro`, or a user attribute. It is a string,
not an `IdentifierNode`, because `derive` and `macro` are reserved words
accepted only in the attribute-name position. `AttributeNode.args` are the
arguments inside the parentheses, if any; the brackets add no AST node.

Remove the `TestDeclaration.title` field:

```typescript
interface TestDeclaration extends BaseNode {
  type: 'TestDeclaration';
  name: IdentifierNode;
  isAsync: boolean;
  body: BlockNode;
}
```

Replace the `TestDeclaration.title` paragraph with:

`TestDeclaration` carries no `title` field. The `#[test]` marker takes no
arguments; `name` is the test's identity for tools, and the note that
`TestDeclaration` carries no `attributes` field is unchanged.

Replace the declaration-attribute sentence in the component note with:

General declaration attributes are the `#[…]` forms — the `AttributeNode`
above — and they appear on no component declaration; neither marker enters
the other's territory ([`types/traits.md`](types/traits.md)).
