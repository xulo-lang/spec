# Design: `#Test` as a declaration attribute

## The decision

The test declaration becomes `TestDecl = "#Test" [ "async" ] "fn"
Identifier "(" ")" Block` — a distinct production whose head token is the
attribute spelling `#Test`. `#Test` is registered in the attribute registry
([types/traits.md](../../types/traits.md)); it is valid only in the test
form, and a `fn` that carries `#Test` but violates the form's shape is
diagnosed (`E0110`). `test` leaves the keyword set and is **not** reserved.

## Alternatives considered

### Keep the `test` keyword (status quo)

**Cost.** The keyword is a word every program loses as an identifier, and it
is the only keyword whose entire job is marking a declaration — it declares
nothing and appears nowhere else. With the attribute registry in the
language, the mark has a home that costs no keyword, and the language keeps
two mechanisms for the same idea. Rejected.

### `#Test` as an attribute on an ordinary `fn`

**Cost.** The test's inertness today rides on the *form*: `TestDeclaration`
binds no name, so no reference can reach the body. If `#Test fn f()` were an
ordinary `FnDeclaration`, `f` would be a binding in the value namespace, and
the inertness (`E0107`) would become a checker carve-out on a declaration
that otherwise binds — the AST type would no longer signal that the name
cannot be referenced, and every consumer of the AST would need the attribute
to know. The distinct production keeps the invariant at the one place it has
always lived: the declaration form itself. Rejected.

## The chosen shape

```text
TestDecl = "#Test" [ "async" ] "fn" Identifier "(" ")" Block ;
```

- The head token is written exactly `#Test`; `async`, when present, follows
  the attribute and precedes `fn` (`#Test async fn`, never
  `async #Test fn`).
- The rest of the form is unchanged: empty parameter list, no return type,
  no generics, no `where`, no `pub`, file scope only.
- The production carries the literal `"#Test"` — the attribute name is part
  of the form, not a free `AttributeList`. The test form therefore accepts
  no other attributes (`#Derive #Test fn` does not parse as a `TestDecl`;
  it parses as an ordinary `fn` carrying two attributes, and `Test` on a
  plain `fn` is `E0110`).

### Grammar ambiguity

`#Test fn f()` (empty parens, no return type, no generics) matches both
`TestDecl` and `FnDecl` with an `AttributeList`. The resolution: when the
token sequence matches the `TestDecl` production, the declaration **is** a
`TestDecl` — the attribute-name lookup never runs. A `#Test` that precedes a
`fn` with parameters, a return type, generics, a `where` clause, or `pub`
does not match `TestDecl`, parses as an ordinary `FnDecl` carrying the
attribute, and is diagnosed `E0110` (the registry entry states the form).

### Registry entry

The registry gains a `Test` entry alongside `Derive`:

- **Placement.** `#Test` is valid only on the test form. Written on
  `struct`, `enum`, `trait`, or `impl`, it is `E0109`. Written on a `fn`
  whose shape does not match the test form (parameters, return type,
  generics, `where`, or `pub`), it is `E0110`.
- **Shape.** The entry's shape is the test form itself: `#Test [async] fn
  name() { … }`, with no parameters and no return type.
- The sentence "Attributes are written on the five item declarations … and
  on nothing else" is extended to include the test form.

### Identifier, keyword, and AST

- The identifier binds no value — any reference is `E0107`, message shape
  unchanged (`` `helper_check` is a test function and cannot be referenced ``).
- `test` becomes an ordinary identifier, **not** reserved — the same fate
  `from` and `import` received when the import syntax was retired. The old
  spelling `test fn …` is a parse error, not a reinterpretation.
- The AST `TestDeclaration` node is unchanged: the node type carries
  test-ness exactly as the keyword did. No attribute field is added — the
  `#Test` token is the production's head, not a carried attribute list.

## Diagnostic consequences

| Code | Change |
|------|--------|
| `E0107` | Trigger re-worded: a reference to the name of a `#Test` function. Message shape unchanged. |
| `E0109` | New instance: `#Test` on `struct`/`enum`/`trait`/`impl`. Message shape follows the existing pattern (`` attribute `Test` is not valid on a struct ``). |
| `E0110` | New instance: `#Test` before a `fn` that does not match the test shape. Message shape: `` attribute `Test` takes the form `#Test [async] fn name() { … }` ``. |
| `E0501` | Trigger re-worded: `await` in a `#Test` function that is not marked `async`. Message shape unchanged. |
