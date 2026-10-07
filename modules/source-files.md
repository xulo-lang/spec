# Source Files

A Xulo module is a file, so the shape of a module is the shape of a file: this
chapter defines the encoding and extension of a source file, the complete set
of declarations that may appear at file scope, the order in which they may
appear, the entry point of an executable program, and how file scope differs
from block scope.

## File format

- A source file uses the extension `.xulo` and MUST be encoded in UTF-8
  ([README §3](../README.md)); encoding, whitespace, statement termination, and
  comment syntax are specified in
  [`../lexical-structure.md`](../lexical-structure.md).
- One file is one module ([README.md](README.md)); the file's contents are the
  whole module, and there is no second unit of module organization — no
  directory module, no module block, and no module header declaration.
- A file is a sequence of top-level declarations. The grammar is
  newline-insensitive and `;` is an optional statement terminator, so
  declarations are delimited by the token rules of
  [`../grammar.md`](../grammar.md), never by layout.

## Top-level declarations

The following forms, and only these, MAY appear at file scope:

| Form | Example | Notes |
|------|---------|-------|
| Function | `fn f(x: int): int` | A function whose return type is `View` is a component |
| Struct | `struct User { … }` | Introduces a nominal type |
| Enum | `enum Theme { … }` | Introduces a nominal type and its variants |
| Trait | `trait Area { … }` | Signature-only capability declaration |
| Impl | `impl Area for Rect { … }` | Attaches methods to a type; never a name of its own |
| Type alias | `type Name = …` | Transparent alias |
| Binding | `let x = 1`, `let mut y = 2` | File-scope variables |
| Constant | `const PI = 3.14` | Compile-time constant |
| Import | `import { add } from "math"` | Binds names from another module |
| Re-export | `pub use { add }` | Adds existing names to the module's interface |

Order rules:

1. **Import declarations MUST appear before any other top-level declaration.**
   All `import` forms together form the file's header: a file that contains an
   `import` anywhere but in its header is ill-formed. One ordering rule then
   covers every file, and a file's dependencies are readable at its head
   ([import-export.md](import-export.md)).
2. `pub use` is an ordinary top-level declaration and MAY appear anywhere
   among the declarations (see [import-export.md](import-export.md)).
3. Every other form MAY appear in any order, subject only to the
   forward-reference rules below.
4. An `import` MUST appear at file scope: it MUST NOT appear inside a function
   body, a block, or a component body.

```xulo
import { add, PI } from "math"     // header
import * as m from "util"
import type { User } from "types"

pub struct Account { owner: User, balance: int }

pub fn deposit(a: Account, amount: int): Account {
  Account(owner: a.owner, balance: a.balance + amount)
}
```

## Declaration order and forward references

Declarations fall into two classes with different ordering rules:

- Module-level `fn`, `struct`, `enum`, `trait`, `type`, and `impl`
  declarations MAY reference each other regardless of the order in which they
  appear in the file. Recursion, mutual recursion, and forward references are
  all well-formed, and a type may be used before the `struct` that declares it.
- Module-level `let`, `let mut`, and `const` MUST be declared before they are
  used. Their initializers run in declaration order, so a use that precedes the
  declaration has nothing to read and MUST be an error.

```xulo
// forward references: fine, whichever order these appear in
fn area(r: Rectangle): float { r.w * r.h }

struct Rectangle { w: float, h: float }

fn describe(r: Rectangle): string {
  `${name()} rectangle`
}

fn name(): string { "large" }
```

```xulo
fn total(): int { running + 1 }   // error: `running` used before its declaration
let running = 0
```

Inside a function body the stricter rule always applies: a local name is in
scope from the end of its declaration to the end of the enclosing block, and
locals are not hoisted ([`../names.md`](../names.md)).

## Entry point

An executable program is a package together with a distinguished *entry
module* — the file from which execution begins. The entry module MUST declare
a module-level function named `main`; every other module is a library module
and declares no entry point.

Allowed signatures:

```xulo
fn main() { … }            // ordinary program: returns unit
fn main(): View { … }       // renderable program: returns View
async fn main() { … }       // asynchronous program: returns Task<unit>
```

- `main` MUST take no parameters and MUST be declared at file scope.
- Its declared return type MUST be omitted (which means `unit`) or MUST be
  `View`; the two signatures are the headless and the renderable program of
  [`../components/view-syntax.md`](../components/view-syntax.md).
- `main` MAY be declared `async`, in which case its declared return type
  denotes `Task<unit>` or `Task<View>` as usual
  ([`../expressions/async-expressions.md`](../expressions/async-expressions.md)).
- `main` MAY be marked `pub`.
- A program whose entry module declares no `main` is rejected, and a `main`
  with parameters or with any other return type is rejected; both are
  diagnostics of the modules family ([`../type-system/errors.md`](../type-system/errors.md)).

## File scope vs block scope

**File scope** is the scope of the whole file. It holds the module's
declarations — `fn`, `struct`, `enum`, `trait`, `impl`, `type`, `let`,
`const` — together with the bindings created by `import` and `pub use`.
Declarations other than `let` and `const` are in scope throughout the entire
file; `let` and `const` are in scope from their declaration to the end of the
file. Everything in file scope that is `pub` forms the module's interface.

**Block scope** is the scope of a `{ … }` body. It holds the body's local
bindings (`let`, `let mut`, `const`), parameters, loop variables, `match` arm
bindings, `catch` bindings, and any **nested `fn`**:

```xulo
fn outer(n: int): int {
  fn double(v: int): int { v * 2 }   // nested function: a local declaration
  double(n) + double(n)
}
```

A nested `fn` is an ordinary local declaration: it is in scope within its own
body, so self-recursion is legal, and from the end of its declaration to the
end of the enclosing block; two nested functions do not forward-reference each
other ([`../functions.md`](../functions.md)). It MAY be referenced by a closure
in the same block, it is never visible outside its module, and it is not part
of the module's interface — a nested function MUST NOT be marked `pub`.
`struct`, `enum`, `trait`, `type`, and `impl` are file-scope declarations: they
introduce the module's types and implementations and are never local to a
block.

A component body is an ordinary block with one extra placement rule: `@State`,
`@Store`, `@Effect`, and `@Environment` are valid only at the top level of a
component body (a `fn` whose declared return type is `View`). See
[`../components/README.md`](../components/README.md).

## Comments

Comments are specified in [`../lexical-structure.md`](../lexical-structure.md):
`//` to the end of the line and `/* … */` blocks that do not nest. Comments
are discarded before parsing, so they are **not part of the module interface**
in any way — a comment exports nothing, documents nothing to the language, and
no rule of this specification reads one. There is no documentation-comment
syntax, indentation is not significant, and formatting choices are therefore
invisible to checking; a file means what its tokens mean, however it is
laid out.
