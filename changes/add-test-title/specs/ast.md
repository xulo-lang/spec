# AST Schema — Delta: Test Title

> **Delta** — adds a `title` field to `TestDeclaration` in `ast.md`.

The interface replaces the current one:

```typescript
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
