# Visibility — Delta: Macro Definition Attributes

> **Delta** — replaces the “`#[macro]` function” paragraph and its examples,
> and the macro bullet of the package-boundary rule of
> `modules/visibility.md`. The remaining visibility rules are unchanged.

## `pub` on declarations

Replace the `#[macro]` function paragraph and its examples with:

A macro implementation function follows the ordinary rule, with one addition:
`pub` exports **two** names — the implementation function's identifier in the
value namespace and the macro name in the macro namespace. Without `pub`, both
are module-private.

```xulo
#[macro_derive(Serialize)]
pub fn derive_serialize(input: ast::DeriveInput): Result<…> { … }
// exports `derive_serialize` and the macro `Serialize`

#[macro_attr]
pub fn with_banner(input: ast::AttributeInput): Result<…> { … }
// exports `with_banner` and the attribute macro `with_banner`

#[macro]
fn sql(input: ast::FunctionInput): Result<…> { … }
// private: neither `sql` nor the function macro `sql` leaves this module
```

The two exports share one visibility: there is no spelling that exports the
macro but keeps the implementation private, or the reverse. A client that
imports the macro name receives the macro; a client that imports the function
name receives the value. For `#[macro]` and `#[macro_attr]` the two names are
the same spelling; for `#[macro_derive]` the function keeps its identifier and
the macro is named by `Target`.

## Package boundaries

Replace the macro bullet with:

- **A macro expands only if its definition is visible at the invocation
  site.** A private macro is not exported: a `use` of it is `E0602`, exactly
  as a `use` of a private function is, and so is a qualified function-macro
  path `m::route!(…)` that reaches it. A private derive or attribute macro is
  not in the importing module's macro namespace, so `#[derive(Serialize)]` or
  `#[route(…)]` that names it is `E1005` or `E0108`; the name was never
  exported.
