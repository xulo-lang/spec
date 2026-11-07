# AST Schema — Delta: Test Options

> **Delta** — replaces `AttributeNode`, adds `AttributeArgNode` and
> `AttributeValueNode`, adds `TestMode`, adds `mode`, `expectedMessage`,
> `isSkipped`, `skipReason`, and `timeoutMs` to `TestDeclaration`, and
> replaces the `AttributeNode` and `TestDeclaration` notes in `ast.md`. No
> other node changes.

## Attribute arguments

Replace `AttributeNode` with:

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: AttributeNameNode;
  args: AttributeArgNode[] | null;
}

interface AttributeArgNode extends BaseNode {
  type: 'AttributeArg';
  name: IdentifierNode | null;
  value: AttributeValueNode;
}

type AttributeValueNode = IdentifierNode | StringLiteral | NumberLiteral;
```

Replace the sentence about `AttributeNode.args` in the attribute note with:

`AttributeNode.args` is `null` for a bare attribute and an array when a
parenthesized group is written; the array is empty for `#[test()]`. An
`AttributeArgNode` with a non-null `name` is the `name = value` form; a bare
identifier argument has `name: null` and an `IdentifierNode` value. In this
change `AttributeValueNode` is an identifier, string, or integer literal;
`add-macro-system` extends it with positional values, floats, and booleans.

## Test declaration

Replace `TestDeclaration` with:

```typescript
type TestMode = 'normal' | 'shouldFail' | 'shouldPanic';

interface TestDeclaration extends BaseNode {
  type: 'TestDeclaration';
  name: IdentifierNode;
  isAsync: boolean;
  mode: TestMode;
  expectedMessage: StringLiteral | null;
  isSkipped: boolean;
  skipReason: StringLiteral | null;
  timeoutMs: NumberLiteral | null;
  body: BlockNode;
}
```

Replace the `TestDeclaration` note with:

`TestDeclaration` carries no `title` or `attributes` field. `mode` is
`normal` when the marker has no expectation, `shouldFail` for `should_fail`,
and `shouldPanic` for both `should_panic` forms. `expectedMessage` is
non-null only for `should_panic = "…"`; it is the string literal written as
the expected substring, including its span and cooked text.

`isSkipped` is true only when `skip` is written; because `skip` is the only
option in its group, a skipped declaration has `mode: 'normal'`,
`expectedMessage: null`, and `timeoutMs: null`. `skipReason` is non-null only
for `skip = "…"` and carries the literal's span and cooked text. `timeoutMs`
is non-null only for `timeout = N`; it is a `NumberLiteral` of kind `integer`
whose value is greater than zero.

The options are metadata of the declaration, not ordinary attributes: they do
not enter name resolution or macro expansion.
