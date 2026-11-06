# Design: Lowercase Attribute Brackets

This document records why declaration attributes become lowercase
`#[name]`, how the reserved words `derive` and `macro` are accepted as
attribute names without becoming identifiers, and what the grammar, AST, and
diagnostics become. The prose that replaces the chapter text is in
[specs/](specs/); the motivation and scope are in [proposal.md](proposal.md).

## Marker form — lowercase `#[…]`

### Brackets, lowercase names, contextual reserved names — CHOSEN

```xulo
#[derive(Debug)]
#[derive(Debug)]          // error[E0805]: the impl would be generated twice
enum Shape { Circle(float), Rect(float, float) }

#[test]
fn absent_key_is_null() { … }

#[test]
fn parses_a_key() { … }

#[macro(kind = "attribute")]
fn route(input: ast::AttributeInput) -> Result<ast::Output, ast::MacroFailure> { … }
```

The `#[macro(kind = …)]` argument shape comes from `add-macro-system`;
this change only re-spells the attribute name.

- **Pro:** the use site is exactly Rust's: `#[derive(Debug)]` and
  `#[test]`, beside `sql!(…)` for function macros. The reader
  does not translate capitalization or add brackets mentally.
- **Pro:** the semantics are unchanged. The registry still decides placement
  and argument shape; derive expansion, test checking, and macro semantics do
  not move. Only the spelling and the attribute-name AST node change.
- **Con:** `derive` and `macro` are reserved words
  ([lexical-structure.md](../../../lexical-structure.md)). The grammar needs one
  small production, `AttributeName`, so those two spellings are accepted in
  the attribute-name position without becoming identifiers anywhere else.
- **Con:** `AttributeNode.name` cannot remain an `IdentifierNode` for
  `derive` and `macro`. It becomes an `AttributeNameNode` carrying the
  spelling as a string.

**Decision.** An attribute is `#`, `[`, an attribute name, an optional
parenthesized argument list, and `]`:

```text
Attribute     = "#" "[" AttributeName [ "(" [ AttributeArgs ] ")" ] "]" ;
AttributeName = Identifier | "derive" | "macro" ;
```

The test declaration has one fixed shape:

```text
TestDecl = "#" "[" "test" "]" [ "async" ] "fn" Identifier "(" ")" Block ;
```

The head is exactly `#[test]`, with no argument group. `#[test()]`,
`#[test("…")]`, and `#[test(name, …)]` do not form a `TestDecl`; on a
non-matching `fn` they are ordinary attribute uses and `E0110`, because the
`test` attribute takes no arguments.

The registry names change with the spelling:

| Old name | New name | Form |
|----------|----------|------|
| `Derive` | `derive` | `#[derive(Debug)]` |
| `Test` | `test` | `#[test]` |
| `Macro` | `macro` | `#[macro(kind = …)]` |
| user attribute | user attribute | `#[route]`, `#[route(…)]` |

Derive targets are not attribute names: `Debug` stays `Debug`.

### Keep `#Name` — REJECTED

- **Pro:** no source churn; the current chapters and the draft
  `add-macro-system` remain as written.
- **Con:** it is the spelling the user has explicitly asked to replace. It
  also leaves the completed macro system with `#route(…)` beside `sql!(…)`,
  two visually unrelated use sites for one feature family.

Rejected. The existing `change-test-to-attribute` chose `#Name` as “Xulo's
own spelling”; this change reverses that spelling decision, not the choice to
express tests as attributes.

### Brackets but keep the capitalized names — REJECTED

- **Pro:** no registry rename and no `AttributeName` production; the AST
  keeps `AttributeNode.name: IdentifierNode`.
- **Con:** it is not the Rust spelling the user asked for. `#[Derive]` still
  has to be translated to `#[derive]` by every reader, and the attribute
  family has one more difference from Rust than necessary.

Rejected. The bracket form and the lowercase names belong together.

### Lowercase by unreserving `derive` and `macro` — REJECTED

- **Pro:** no `AttributeName` production and no new AST node; `#[derive]` and
  `#[macro]` simply use the identifier arm.
- **Con:** it changes the keyword set everywhere. `fn derive()`, `let macro =
  1`, and a field named `macro` would become legal, and the reserved-word
  list would lose two entries for an attribute-only requirement.
- **Con:** a future `macro` declaration syntax would no longer have the
  spelling reserved.

Rejected. The attribute position is the narrow place to admit the two
spellings.

### Lowercase only `test`; keep `Derive` and `Macro` — REJECTED

- **Pro:** `test` is already an ordinary identifier, so only the test form
  changes to the Rust spelling the user named explicitly.
- **Con:** the registry would mix `#[test]` with `#[Derive]` and `#[Macro]`
  for no reason a reader can infer. The request is lowercase names, not one
  lowercase name.

Rejected. All built-in attribute names move together.

### Brackets for the new macro attributes only — REJECTED

- **Pro:** `#[macro]`, `#[route]`, and the other new macro forms read like
  Rust; existing `#[Derive]` and `#[Test]` stay untouched.
- **Con:** the grammar would admit two attribute markers for the same
  registry, and a reader would have to know which attributes are old and
  which are new. A single registry with two spellings is worse than either
  spelling alone.

Rejected. One marker and one spelling for one attribute mechanism.

## Reserved attribute names

`derive` and `macro` remain reserved words. The attribute name is its own
production:

```text
AttributeName = Identifier | "derive" | "macro" ;
```

- In every identifier position, `derive` and `macro` are still rejected as
  before. The exception is only the token between `#[` and `]`.
- `test` is an ordinary identifier, so `#[test]` uses the `Identifier` arm
  and needs no exception.
- Attribute names occupy their own namespace and bind nothing. The AST
  stores the spelling in `AttributeNameNode.name: string`; it is not an
  `IdentifierNode` and does not claim to be an identifier.
- A user attribute is an ordinary identifier and follows the identifier
  rules, so a user attribute cannot be named `derive` or `macro`; those two
  spellings belong to the built-in registry entries.

## No test title

Tests align with the simple `#[test]` marker: the attribute takes no arguments
and `TestDeclaration` loses its `title` field. A test that carried
`#Test("parse test")` becomes `#[test]`; its identifier and body are unchanged
and the title is discarded.

Keeping the title inside the brackets — `#[test("parse test")]` — was rejected
because the requested form has no title. `#[test()]` is likewise not a test
declaration; it is an ordinary attribute use and is `E0110`.

## Lexical consequences

`#` remains one token, but it is an attribute opener only as part of
`#[…]`. The old forms are not tokenized differently; they simply fail to
match the attribute productions:

- `#Derive`, `#Test`, and `#Macro` no longer start an attribute: after `#`
  the grammar requires `[`, so the parser reports a syntax error.
- A `#` that is not followed by `[` is still an error in the parser, not a
  new lexical error. The lexer continues to recognize `#` as a token.
- `[` and `]` are already delimiters; brackets in an attribute are ordinary
  grouping tokens, not a new token class.
- `derive` and `macro` are still reserved tokens. The parser accepts them in
  `AttributeName`; the lexer does not need a new mode.

## AST consequences

`AttributeNode` gains an attribute-name node:

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

- `name.name` is the spelling: `derive`, `test`, `macro`, or a user
  attribute.
- `args` are still the arguments inside the parentheses; the brackets add no
  argument node.
- The attribute's span covers the leading `#` through the closing `]`.
- `TestDeclaration` no longer carries a `title` field. The `#[test]` marker
  takes no arguments, and the note that `TestDeclaration` carries no
  `attributes` field is unchanged.

## Diagnostic consequences

No error code is added or removed. The examples and message shapes that quote
the old spelling change to the lowercase names:

| Code | New example |
|------|-------------|
| `E0107` | `helper_check()` where `#[test] fn helper_check` exists |
| `E0108` | `#[inline]`; message: `` attribute `inline` is not defined `` |
| `E0109` | `#[derive(Debug)] fn main()`; message: `` attribute `derive` is not valid on a function `` |
| `E0110` | `#[derive]`; message: `` attribute `derive` takes a parenthesized list of trait names `` |
| `E0111` | `#[derive(Display)]`; message: `` `Display` is not a registered derive target `` |
| `E0112` | `#[derive(Debug)] struct Box<const N: int> { v: int }` |

`E0110`'s test-form message becomes the bracket shape:

> attribute `test` takes no arguments and is written `#[test]`, followed by
> `[async] fn name() { … }`

Outside a matching `TestDecl`, `#[test]`, `#[test()]`, and `#[test(name, …)]`
are `E0110`; a group that the current `AttributeArg` grammar cannot parse —
notably a `StringLiteral` or a numeral — is a syntax error. When
`add-macro-system` widens `AttributeArg`, such a group parses and falls under
the same `E0110` rule; no title special case remains.

The old spelling is a syntax error and carries no language error code; a
compiler MAY add a migration note. An old capitalized bracket spelling such
as `#[Derive]` parses as an ordinary attribute name and is `E0108` unless a
user attribute of that spelling is in scope.

## Compatibility and migration

Every occurrence is a one-for-one rewrite. There is no parsing ambiguity
because `#` now must be followed by `[`, and no program is silently
reinterpreted: `#Derive` does not become anything else.

The draft `add-macro-system` change is the only dependent proposal. Its
`#Macro`, `#Derive`, `#Test`, user-attribute examples, and its own AST delta
must be rebased before acceptance. Its normative content — input records,
context, execution, quota, cache, sandbox — is independent of the spelling.

## Impact

| Chapter | Change |
|---------|--------|
| [`lexical-structure.md`](../../../lexical-structure.md) | the `#` token's reading becomes “attribute opener, only in `#[…]`”; `derive`/`macro` remain reserved, accepted only in the attribute-name position; the old `#Name` form is called out as invalid |
| [`grammar.md`](../../../grammar.md) | `Attribute` gains brackets, `AttributeName` is added, `TestDecl` uses `#[test]` and drops the title group, and the prose around `#[test]` is re-spelled |
| [`ast.md`](../../../ast.md) | `AttributeNode.name` becomes `AttributeNameNode`; the declaration-attribute note names `#[…]`; attribute spans are stated and the `TestDeclaration` title field is removed |
| [`types/traits.md`](../../../types/traits.md) | the Attributes section and its derive/test examples use `#[derive]` / `#[test]`; the test shape message changes |
| [`testing.md`](../../../testing.md) | the declaration grammar and examples use `#[test]`, and the title description is removed |
| [`type-system/errors.md`](../../../type-system/errors.md) | attribute examples and message shapes use the lowercase names |
| [`type-system/checking-rules.md`](../../../type-system/checking-rules.md) | the test-identifier bullet uses `#[test]` |
| [`modules/source-files.md`](../../../modules/source-files.md) | the test row and forward-reference bullet use `#[test]` |
| [`documentation-comments.md`](../../../documentation-comments.md) | the inert-code bullet and attribute-lines rule use `#[test]` |
| [`builtins/prelude.md`](../../../builtins/prelude.md) | the `Debug` protocol row and paragraph use `#[derive(Debug)]` |
| [`builtins/intrinsic-functions.md`](../../../builtins/intrinsic-functions.md) | the derive example and `assert` sentence use the lowercase forms |
| [`expressions/async-expressions.md`](../../../expressions/async-expressions.md) | the two `await` legality bullets use `#[test]` |
| [`introduction.md`](../../../introduction.md) | the test-framework sentence uses `#[test]` |
| `changes/add-macro-system` | rebase its attribute examples and AST delta; see [tasks.md](tasks.md) |
