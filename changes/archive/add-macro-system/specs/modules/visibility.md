# Visibility — Delta: Macro Visibility

> **Delta** — adds the `#[macro]` case to “`pub` on declarations” and to the
> package-boundary rule of `modules/visibility.md`. The remaining visibility
> rules are unchanged.

## `pub` on declarations

A `#[macro]` function follows the ordinary rule, with one addition: `pub`
exports **two** names — the implementation function's identifier in the value
namespace and the macro name in the macro namespace. Without `pub`, both are
module-private.

```xulo
#[macro(kind = "derive", target = Serialize)]
pub fn derive_serialize(input: ast::DeriveInput): Result<…> { … }
// exports `derive_serialize` and the macro `Serialize`

#[macro(kind = "attribute", name = "route")]
fn route_attribute(input: ast::AttributeInput): Result<…> { … }
// private: neither `route_attribute` nor `route` leaves this module
```

The two exports share one visibility: there is no spelling that exports the
macro but keeps the implementation private, or the reverse. A client that
imports the macro name receives the macro; a client that imports the function
name receives the value.

## Package boundaries

Add to the cross-module rule:

- A macro expands only if its declaration is visible at the invocation site.
  A private macro is not exported: a `use` of it is `E0602`, exactly as a
  `use` of a private function is, and so is a qualified function-macro path
  `m::route!(…)` that reaches it. A private derive or attribute macro is not
  in the importing module's macro namespace, so `#[derive(Serialize)]` or
  `#[route(…)]` that names it is `E1005` or `E0108`; the name was never
  exported.
- A `pub use` of a macro re-exports the macro name under the ordinary
  re-export rule; the macro's implementation function is not re-exported by
  that name unless it was also imported and re-exported as a value.
