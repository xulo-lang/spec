# Traits — Delta: Extensible Attributes and Macro Definitions

> **Delta** — replaces the opening paragraphs and the four registry bullets of
> the Attributes section, adds a “Macro definitions” subsection, and replaces
> the target-registry and generic-type passages of the Derive section of
> `types/traits.md`. The `#Test` and `#Derive(Debug)` rules not mentioned here
> are unchanged.

## Attributes

A declaration MAY carry attributes written directly before it: the name
`#Name`, optionally followed by a parenthesized argument list. The attribute
form is generic; the names the checker interprets are the **built-in
registry** plus the **attribute macros** registered in the current module.

```xulo
#Derive(Debug)
struct User { name: string, age: int }

#route("/v1/users", "GET")
fn get_users() { … }
```

An attribute on its own line, directly above the declaration it marks, is
**RECOMMENDED**; a shared line — `#Derive(Debug) struct User { … }` — is
equally well-formed, the grammar being newline-insensitive
([lexical-structure.md](../lexical-structure.md)).

- The built-in attributes are `Derive`, `Test`, and `Macro`. They keep the
  positions and argument shapes stated below and in
  [`../grammar.md`](../grammar.md). An attribute whose name is not built in
  and not registered by an attribute macro in scope is `E0108`.
- Attribute names occupy their own namespace. `#Derive` binds nothing and
  takes no name away from values, types, or modules. A macro name may equal a
  value or type name in the same module.
- A registered attribute is valid on the declarations its entry allows
  (`E0109`), and its arguments must fit the shape that entry states (`E0110`).
  Built-in attributes keep their existing positions. An attribute macro is
  valid on all eight attribute hosts — `fn`, `struct`, `enum`, `trait`,
  `impl`, `type`, `let`, and `const` — unless its implementation returns an
  `ast::MacroFailure` for a host it does not support.
- The built-in hosts are:

  | Attribute | `fn` | `struct` | `enum` | `trait` | `impl` | `type` | `let` | `const` |
  |-----------|------|----------|--------|---------|--------|--------|-------|---------|
  | `#Derive` | — | yes | yes | — | — | — | — | — |
  | `#Test` | yes | — | — | — | — | — | — | — |
  | `#Macro` | yes | — | — | — | — | — | — | — |
  | user attribute macro | yes | yes | yes | yes | yes | yes | yes | yes |

  `type` is a declaration only; `let` and `const` may be module-level or
  statements. The `#Test` and `#Macro` rows are further restricted by their
  own shape and scope rules.
- There are no inert attributes: every attribute written in source is
  interpreted by a built-in rule or by a macro.
- Attribute arguments are values: an identifier, a string, an integer, a
  float, or a boolean, optionally written `name = value`. A bare identifier in
  a built-in attribute is a name (`#Derive(Debug)`); in an attribute macro it
  is the identifier's spelling, not a value lookup.

### Macro definitions

`#Macro(kind = …, …)` on a module-level `fn` registers a macro; the function
is the implementation and is ordinary checked Xulo code. The shape is fixed by
[`../type-system/checking-rules.md`](../type-system/checking-rules.md):

```xulo
#Macro(kind = "derive", target = Serialize)
fn derive_serialize(
  input: ast::DeriveInput
): Result<ast::DeriveOutput, ast::MacroFailure> { … }

#Macro(kind = "attribute", name = "route")
fn route_attribute(
  input: ast::AttributeInput
): Result<ast::AttributeOutput, ast::MacroFailure> { … }

#Macro(kind = "function", name = "sql")
fn sql_macro(
  input: ast::FunctionInput
): Result<ast::FunctionOutput, ast::MacroFailure> { … }
```

- The macro name is the `target` identifier for a derive macro and the `name`
  string for an attribute or function macro. The implementation function's
  identifier is independent.
- Important: the macro's name is fully qualified by its module. `m1::w` and
  `m2::w` are different macros; a short name requires `use`.
- `pub` on the implementation function exports the macro together with the
  function value; without `pub`, both are module-private
  ([`../modules/visibility.md`](../modules/visibility.md)).
- A macro name MUST be a legal identifier and MUST NOT be a keyword or a
  reserved word. It MUST NOT be a built-in attribute name (`Derive`, `Test`,
  `Macro`); a derive macro MUST NOT also use a reserved derive target
  (`Debug`, `Display`, `ToString`). Two macros with one name in one module are
  `E1010`; a malformed `#Macro` is `E1010`.
- An attribute macro replaces the decorated declaration (its
  `ast::AttributeOutput.item` is required and must be a declaration of the same
  host kind) and may add items beside it. It cannot delete the declaration.
- Macros expand before global name resolution and type checking; the expanded
  AST is checked exactly like written code
  ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).

## Derive

`#Derive(Debug)` on a `struct` or an `enum` generates exactly the impl an
author would write by hand for the structural case. The body is fixed by the
target:

The registry of derive targets contains the built-in target `Debug` plus every
user derive macro in scope. `Display` and `ToString` remain reserved and are
not registered; written as targets they are `E0111`. A target that is neither
built in nor a derive macro in scope is `E1005`.

Replace the generic-type bullet of the Derive rules with:

- `#Derive(Debug)` on a type that declares generic parameters is `E0112`: the
  language has no bound that means “is not a `struct`, or implements
  `Debug`”, so the built-in expansion cannot be written for an open parameter.
  A user derive macro is not subject to this restriction: it receives the
  generic parameters in `ast::DeriveInput.target.generics` and chooses what to
  generate. A generic type keeps its hand-written impl when no macro handles
  it.

Add to the Derive rules:

- A user derive macro receives one `ast::DeriveInput` and returns
  `ast::DeriveOutput`; its `items` are inserted after the decorated
  declaration. The generated items are checked as ordinary items.
- Derive targets expand from left to right before any attribute macro on the
  same declaration expands.
- A generated impl that collides with a written impl, or with another
  generated impl, is the ordinary conflicting-impl error `E0805` when the
  expanded AST is checked.
