# AST Schema — Delta: Test Options

> **Delta** — adds `TestMode`, adds `mode`, `expectedMessage`, `isSkipped`,
> `skipReason`, and `timeoutMs` to `TestDeclaration`, and replaces the
> `TestDeclaration` note in `ast.md`. `AttributeNode`, `AttributeArgNode`, and
> `AttributeValueNode` are unchanged: the accepted `add-macro-system` change
> already supplies them. No other node changes.

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
