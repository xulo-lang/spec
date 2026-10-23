# Source Files — Delta: Exit Status Cross-Reference

> **Delta** — adds one bullet to the `main` bullet list in the
> “The root module and `main`” section of `modules/source-files.md`, after
> the “`main` MAY be marked `pub`” bullet; the bullets before and after it
> are unchanged.

## The root module and `main`

An executable program is a module tree whose root module declares a
module-level function named `main`; every other module of the tree is a
library module and declares no entry point.

Allowed signatures:

```xulo
fn main() { … }            // ordinary program: returns unit
fn main(): View { … }       // renderable program: returns View
async fn main() { … }       // asynchronous program: returns Task<unit>
```

- `main` MUST take no parameters and MUST be declared at the root module's
  file scope.
- Its declared return type MUST be omitted (which means `unit`) or MUST be
  `View`; the two signatures are the headless and the renderable program of
  [`../components/view-syntax.md`](../components/view-syntax.md).
- `main` MAY be declared `async`, in which case its declared return type
  denotes `Task<unit>` or `Task<View>` as usual
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
- `main` MAY be marked `pub`.
- A program that terminates normally exits with status 0, and one that ends
  in a runtime failure exits with a nonzero status; the rule is specified
  in [`../memory-and-runtime.md`](../memory-and-runtime.md).
- A program whose root module declares no `main` is `E0608`, and a `main`
  with parameters or with any return type other than the two above is
  `E0609`; both are modules-family diagnostics
  ([`../type-system/errors.md`](../type-system/errors.md)).
