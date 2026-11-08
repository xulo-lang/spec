# Names — Delta: Macro Definition Attributes

> **Delta** — replaces the macro row of the named-entities table, the macro
> scope row, the declaration-order bullet, and the macro-resolution paragraph
> of `names.md`. The other name rules are unchanged.

## Identifiers and Named Entities

Replace the macro row with:

| Macros | `#[macro]`, `#[macro_attr]`, or `#[macro_derive(Target)]` on a module-level `fn`; `use` bindings | The macro name is the function identifier for `#[macro]`/`#[macro_attr]`, and `Target` for `#[macro_derive]`; macro names form their own namespace |

## Scopes

Replace the macro scope row with:

| Macro scope | A module-level macro definition attribute; a macro `use` or `pub use` binding | The macro's fully qualified name throughout its module; a short macro name throughout the importing module |

Replace the declaration-order bullet with:

> - **At module level**, a macro definition attribute is registered before any
>   expansion, so its macro name is visible throughout the whole file,
>   whichever order it appears in. Macro names are a separate namespace: they
>   do not collide with value or type names, and two declarations with one
>   macro name in one module are `E1010` rather than shadowing.

Replace the macro-resolution paragraph with:

A macro name resolves in the macro namespace — the module's own macro
definitions, then macro `use` bindings — and only at a macro position
(`#[derive(T)]`, `#[name(…)]`, `name!(…)`). There are no local macro bindings. In
a value position the macro name is not a value and is `E0101`; the
implementation function's own identifier is the value. For `#[macro]` and
`#[macro_attr]` the function identifier is both the value name and the macro
name; for `#[macro_derive(Target)]` the implementation function identifier is
independent of `Target`.
