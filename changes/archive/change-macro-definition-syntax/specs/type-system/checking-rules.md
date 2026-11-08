# Checking Rules — Delta: Macro Definition Attributes

> **Delta** — replaces the definition-pre-pass wording, the “Macro
> definitions” section, the kind-selection sentence of invocation resolution,
> and the duplicate/order rules of `type-system/checking-rules.md`. The
> expansion loop and generated-code rules are unchanged.

## Expansion pass

In step 2, replace “every `#[macro]` definition” with “every function carrying
a macro definition attribute”. In step 3, replace “every `#[macro]`
definition written in the module” with “every function carrying a macro
definition attribute written in the module”.

## Macro definitions

Replace the `#[macro]` definitions section with:

A function is a macro implementation when it carries exactly one of the
built-in definition attributes `#[macro]`, `#[macro_attr]`, or
`#[macro_derive(Target)]`. Those attributes are registered on `fn` only; on
any other host each is the ordinary attribute-host error `E0109`.

Exactly one definition attribute MAY appear on a function. Multiple definition
attributes are `E1010`. The definition function MUST NOT also carry a user
attribute macro; registration happens before expansion, so such an attribute
would replace the function after the macro's implementation was registered,
and the registered implementation and the checked function would diverge.
Other attributes follow their own host rules.

Macro definitions are collected from the written AST before expansion; a macro
definition attribute produced by an expansion is not registered and is
`E1010`.

On a `fn` the definition is valid only when all of the following hold. A
violation of this list is `E1010`:

- It is not `async` and not `extern`, declares no generic parameters, and has
  no `where` clause.
- It has exactly one parameter, with no mode and no default.
- Its parameter type and return type match the chosen attribute:

| Attribute | Parameter | Return |
|-----------|-----------|--------|
| `#[macro]` | `ast::FunctionInput` | `Result<ast::FunctionOutput, ast::MacroFailure>` |
| `#[macro_attr]` | `ast::AttributeInput` | `Result<ast::AttributeOutput, ast::MacroFailure>` |
| `#[macro_derive(Target)]` | `ast::DeriveInput` | `Result<ast::DeriveOutput, ast::MacroFailure>` |

- `#[macro]` and `#[macro_attr]` take no positional arguments.
  `#[macro_derive]` takes exactly one positional identifier, its target; that
  target is a legal identifier and is not a keyword or reserved word.
- `max_steps`, `max_memory`, and `max_depth`, when written, are non-negative
  integer literals; no other argument is accepted.
- The macro name is the function identifier for `#[macro]` and
  `#[macro_attr]`, and the target identifier for `#[macro_derive]`. It is not
  already registered in the module's macro namespace, is not a built-in
  attribute name (`derive`, `test`, `macro`, `macro_attr`, `macro_derive`),
  and, for `#[macro_derive]`, is not `Debug`, `Display`, or `ToString`.
- The declaration is not inside a function body.
- Its body contains no user macro invocation (`name!(…)`); a macro-to-macro
  call uses an `ast::call_*` entry point.

The old `#[macro(kind = …)]` form is removed: `#[macro]` now takes no
positional arguments, so a `kind` argument is a malformed `#[macro]`
definition (`E1010`) whose note names the matching new attribute.

## Invocation resolution

Add this sentence before the bullet list:

The kind of a macro is the one selected by its definition attribute
(`#[macro]`, `#[macro_attr]`, or `#[macro_derive(Target)]`); no `kind` key is
written.

In the Attribute bullet, replace “A built-in attribute name (`derive`, `test`,
`macro`)” with “A built-in attribute name (`derive`, `test`, `macro`,
`macro_attr`, `macro_derive`)”.

## Duplicate and order rules

Replace the first three bullets with:

- Two macro definitions with one macro name in one module are `E1010`. For
  `#[macro]` and `#[macro_attr]`, the macro name is the function identifier,
  so a duplicate identifier is reported once by the ordinary
  duplicate-declaration rule at the function level, not as a second macro-name
  error. For `#[macro_derive]`, two functions with different identifiers may
  target the same name, which is `E1010`.
- Two macros of different kinds with one name are also `E1010`: the macro
  namespace is one namespace.
- A macro may share a spelling with a value or type in the same module; for
  `#[macro]` and `#[macro_attr]`, the function value intentionally shares the
  spelling with the macro.
