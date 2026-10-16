# Source Files — Delta: Declaration-Built Modules

> **Delta** — replaces the **File format**, **Top-level declarations**,
> **Entry point**, and **File scope vs block scope** sections of
> `modules/source-files.md`. The **Declaration order and forward
> references** and **Comments** sections are unchanged.

## File format

- A source file uses the extension `.xulo` and MUST be encoded in UTF-8
  ([README §3](../README.md)); encoding, whitespace, statement termination,
  and comment syntax are specified in
  [`../lexical-structure.md`](../lexical-structure.md).
- A file is a sequence of top-level content: an optional **header** of
  module declarations, then declarations. The grammar is newline-insensitive
  and `;` is an optional statement terminator for declarations, so
  statements are delimited by the token rules of
  [`../grammar.md`](../grammar.md), never by layout; the header forms are
  terminated by `;` (an inline `mod` block is its own delimiter).
- A file becomes a module when a `mod` declaration claims it: one file may
  be the body of at most one module, several modules may live in one file as
  inline `mod` blocks, and a file no declaration claims is not part of the
  package ([README.md](README.md), [import-export.md](import-export.md)).

## Top-level declarations

The following forms, and only these, MAY appear at file scope:

| Form | Example | Notes |
|------|---------|-------|
| Module declaration | `mod form;`, `mod tests { … }` | Claims a file, or nests a module in this one; `pub mod` exports the namespace |
| Use | `use form::validate;` | Binds names of a path in this file's scope |
| Re-export | `pub use ui::HStack;` | Binds a path and adds it to the module's interface |
| Function | `fn f(x: Int): Int` | A function whose return type is `View` is a component |
| Struct | `struct User { … }` | Introduces a nominal type |
| Enum | `enum Theme { … }` | Introduces a nominal type and its variants |
| Trait | `trait Area { … }` | Signature-only capability declaration |
| Impl | `impl Area for Rect { … }` | Attaches methods to a type; never a name of its own |
| Type alias | `type Name = …` | Transparent alias |
| Binding | `let x = 1`, `let mut y = 2` | File-scope variables |
| Constant | `const PI = 3.14` | Compile-time constant |
| Test | `test fn check() { … }` | Checked like a function body; run only by a test tool ([testing.md](../testing.md)) |

Order rules:

1. **The header comes first.** The header consists of every `mod`, `use`,
   and `pub use` form at this level; all three MAY appear in any
   order among themselves, and the first form of any other kind ends the
   header. A header form written after a top-level declaration is `E0605`.
   One rule then covers every file: dependencies and structure are readable
   at its head ([import-export.md](import-export.md)).
2. **Header forms appear only in a header.** They MUST NOT appear inside a
   function body, a plain block, or a component body; the header of an
   inline `mod` block is that module's, not the enclosing function's.
3. Every other form MAY appear in any order, subject only to the
   forward-reference rules below.

```xulo
mod form;
mod internal;

use lsp-server::router::Router;
use ui::{HStack, Button};
use type ui::User;
pub use ui::hstack::HStack;

pub struct Account { owner: User, balance: Int }

pub fn deposit(a: Account, amount: Int): Int { a.balance }
```

## Entry point

An executable program is a package whose **root module** — the file the
environment designates, from which the `mod` tree grows — declares a
module-level function named `main`; every other module of the tree is a
library module and declares no entry point.

Allowed signatures:

```xulo
fn main() { … }            // ordinary program: returns Unit
fn main(): View { … }       // renderable program: returns View
async fn main() { … }       // asynchronous program: returns Task<Unit>
```

- `main` MUST take no parameters and MUST be declared at the root module's
  file scope.
- Its declared return type MUST be omitted (which means `Unit`) or MUST be
  `View`; the two signatures are the headless and the renderable program of
  [`../components/view-syntax.md`](../components/view-syntax.md).
- `main` MAY be declared `async`, in which case its declared return type
  denotes `Task<Unit>` or `Task<View>` as usual
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
- `main` MAY be marked `pub`.
- A program whose root module declares no `main` is `E0608`, and a `main`
  with parameters or with any return type other than the two above is
  `E0609`; both are modules-family diagnostics
  ([`../type-system/errors.md`](../type-system/errors.md)).

## File scope vs block scope

**File scope** is the scope of the whole module body — the file's contents,
or an inline `mod` block's contents. It holds the module's declarations —
`fn`, `struct`, `enum`, `trait`, `impl`, `type`, `let`, `const`, `mod` —
together with the bindings created by `use` and `pub use`. Declarations
other than `let` and `const` are in scope throughout the entire body; `let`
and `const` are in scope from their declaration to the end of the body.
Everything in file scope that is `pub` — declarations, `pub mod` names, and
`pub use` entries — forms the module's interface.

**Block scope** is the scope of a `{ … }` body. It holds the body's local
bindings (`let`, `let mut`, `const`), parameters, loop variables, `match`
arm bindings, and any **nested `fn`**:

```xulo
fn outer(n: Int): Int {
  fn double(v: Int): Int { v * 2 }   // nested function: a local declaration
  double(n) + double(n)
}
```

A nested `fn` is an ordinary local declaration: it is in scope within its own
body, so self-recursion is legal, and from the end of its declaration to the
end of the enclosing block; two nested functions do not forward-reference each
other ([`../functions.md`](../functions.md)). It MAY be referenced by a closure
in the same block, it is never visible outside its module, and it is not part
of the module's interface — a nested function MUST NOT be marked `pub`.
`struct`, `enum`, `trait`, `type`, `impl`, and `mod` are file-scope
declarations: they introduce the module's types, implementations, and
submodules and are never local to a block.

A component body is an ordinary block with one extra placement rule: `@State`,
`@Store`, `@Effect`, and `@Environment` are valid only at the top level of a
component body (a `fn` whose declared return type is `View`). See
[`../components/README.md`](../components/README.md).
