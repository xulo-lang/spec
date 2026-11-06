# AST Schema — Delta: Rust-Style Attributes

> **Delta** — replaces `AttributeNode`, adds `AttributeNameNode`, adds a
> source note after them, replaces the `TestDeclaration.title` paragraph,
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

Replace the `TestDeclaration.title` paragraph with:

`title` is `null` exactly when the attribute's parenthesized group is absent
or empty — `#[test]` and `#[test()]` both mean no title. It is a dedicated
field, not an `attributes` entry, and the note that `TestDeclaration` carries
no `attributes` field is unchanged. The title is written inside the attribute
brackets — `#[test("parse test")]` — and is metadata for reporting: it is not
a name, and `TestDeclaration.name` keeps every rule it has.

Replace the declaration-attribute sentence in the component note with:

General declaration attributes are the `#[…]` forms — the `AttributeNode`
above — and they appear on no component declaration; neither marker enters
the other's territory ([`types/traits.md`](types/traits.md)).
