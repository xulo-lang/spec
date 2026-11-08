# AST — Delta: Macro Definition Attributes

> **Delta** — replaces the attribute-argument note and the macro-definition
> note of `ast.md`. The node shapes are unchanged.

## Declarations

Replace the attribute note's sentence about `AttributeArgNode` with:

An `AttributeArgNode` with a non-null `name` is the `name = value` form, which
the definition attributes use (`#[macro_derive(Serialize)]`); a positional
argument has `name: null` (`#[derive(Debug)]`, `#[route("/v1/users")]`).

Replace the macro-definition note with:

A macro definition attribute (`#[macro]`, `#[macro_attr]`, or
`#[macro_derive(Target)]`) adds no declaration node: it is an ordinary
`FnDeclaration` whose `attributes` contains the corresponding `AttributeNode`.
The same is true of `#[derive]` and `#[test]` on their existing nodes.
