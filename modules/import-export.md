# Import and Export

Names cross module boundaries through two mechanisms only: `import` brings
names of another module into this file, and `pub` on a declaration — alone or
with `pub use` — puts a name into this module's interface. This chapter
defines the forms, what each one binds, how a specifier becomes a file, and
the errors that follow from breaking these rules.

## Import forms

There are exactly four import forms. All of them appear in the file's import
header, before any other top-level declaration
([source-files.md](source-files.md)), and all of them name a module by a
string specifier.

### Named import

```xulo
import { add, PI } from "math"
import { add as plus } from "math"
```

A named import binds each listed name as an ordinary file-scope binding of
this module, holding exactly what the target module exports under that name —
a function keeps its signature, a type keeps its declaration, a constant keeps
its type. The bindings take part in name resolution like any other file-scope
name ([`../names.md`](../names.md)), so they may be called, used in
annotations, and used as the base of a member access.

Every entry MUST name a member that the target module exports; a name that
does not exist there is an unknown export, and a name that exists but is not
`pub` is a visibility violation. `as` renames the local binding only — the
right-hand side of `as` is a fresh local name and never affects the target
module:

```xulo
import { add as plus, PI as PI314 } from "math"
plus(1, 2)          // calls `add`
```

### Namespace import

```xulo
import * as math from "math"
```

A namespace import binds one name — here `math` — as a namespace. Its members
are the target module's exports, selected with `.`:

```xulo
print(math.add(2, 3))
print(math.PI)
```

Only exported names are members of the namespace: selecting a private name of
the target module is a visibility violation, and selecting a name the target
module does not have at all is an unknown member. A namespace binding is an
ordinary file-scope name; it MAY be passed around only through `pub use` (a
namespace is not a value).

### Type-only import

```xulo
import type { User, Status } from "types"
```

A type-only import binds the listed names for **type positions only**: type
annotations, type arguments, trait bounds, return types, and the base of an
enum-variant path. It creates no value binding, so the name has no runtime
presence in this module (see [Type-only imports and erasure](#type-only-imports-and-erasure)
below).

The `type` modifier follows the `import` keyword; it is not written inside the
braces. `import { type User } from "types"` is not a form, and using a
type-only binding where a value is expected is an error.

### Side-effect import

```xulo
import "core"
```

A side-effect import binds nothing. It exists so that the target module's
file-scope initializers run before those of the importing module: modules are
initialized in dependency order, and this form forces an edge in that order
without bringing in any name.

## Export forms

Two families of forms put names into a module's interface.

**Declarations marked `pub`.** Every declaration form carries its own `pub`:

```xulo
pub fn add(a: Int, b: Int): Int { a + b }
pub struct User { pub name: String, age: Int }
pub enum Status { Active, Inactive }
pub trait Shape { fn area(self): Float }
pub type Score = Int
pub const PI = 3.14
pub let appName = "xulo"         // a file-scope binding may be exported too
```

**`pub use`.** A re-export adds names to the interface without repeating
`pub` on their declarations:

```xulo
pub use { add, PI }              // names of this module (declared or imported)
pub use { User, Status } from "types"   // names read from another module
```

- `pub use { a, b }` names bindings of *this* module. Each name MUST already
  be part of the interface: either a declaration here that is marked `pub`, or
  a name brought in by an `import` (which was necessarily exported by its
  source). Re-exporting a private local declaration, or naming a binding that
  does not exist, is an error.
- `pub use { x } from "m"` reads `x` from module `m`. It MUST be an export of
  `m`, exactly as if it had been imported by name. This form does **not** bind
  `x` locally: it is a pure forwarding edge, and `x` is usable in this file
  only if the file also imports it.

The complete mapping from declaration to exported name:

| Declaration | Exported name | What importers receive |
|-------------|---------------|------------------------|
| `pub fn f(…)` | `f` | a function of the declared signature |
| `pub struct S { … }` | `S` | the type, its constructor form, and its `pub` members |
| `pub enum E { … }` | `E` | the type, its variants `E::V`, and its `pub` members |
| `pub trait T { … }` | `T` | the trait, to be implemented locally |
| `pub type A = …` | `A` | the transparent alias |
| `pub let x` / `pub let mut x` | `x` | the binding and its declared type |
| `pub const K` | `K` | the constant and its declared type |
| `pub use { a, b }` | `a`, `b` | the interface names of this module |
| `pub use { x } from "m"` | `x` | the same name as exported by `m` |
| `impl …` | — | never exported; an `impl` has no name |

A declaration without `pub` exports nothing, and `impl` blocks are never part
of an interface: importing a type or a trait does not import any `impl`
([`../types/traits.md`](../types/traits.md)).

## Specifiers and resolution

A specifier is a string literal. It is resolved to exactly one module file,
and resolution MUST be unambiguous.

**Relative specifiers** start with `./` or `../` and are resolved against the
directory of the importing file:

```xulo
import { area } from "./shapes"        // ./shapes.xulo
import { fmt } from "../lib/util"      // ../lib/util.xulo
```

- The path is normalized (`.` and `..` segments apply as usual).
- The extension `.xulo` is implied and MUST NOT be written: `./shapes` names
  the file `shapes.xulo`.
- A specifier resolves to a **file**. There are no directory modules: `./lib`
  resolves to `lib.xulo`, never to `lib/mod.xulo` or `lib/index.xulo`.
- A relative specifier that does not resolve to an existing file is an error.

**Package specifiers** are everything else — a specifier with no `./` or
`../` prefix:

```xulo
import { hstack } from "@xulo/ui"
import { add } from "math"
```

A package specifier is resolved by the *package environment*: the environment
maps the specifier to one module of the program's packages, and that mapping
is determined outside this specification ([README §1.1](../README.md)). What
this specification requires is only the shape of the rule: a package specifier
MUST resolve to exactly one module, resolution MUST NOT depend on which file
asks, and two different specifiers MAY name the same module — they are then
two ways of reaching one module, which is checked once.

Resolution failures — a specifier that resolves to nothing, to a path that is
not a file, or to more than one candidate — are compile-time errors.

## Cycles

The dependency graph has one edge for each `import` and each
`pub use { … } from "…"`. A cycle in this graph — direct (`a` imports `a`),
of length two (`a` imports `b`, `b` imports `a`), or longer — is a
compile-time error, and the error MUST name the cycle path in order, for
example `a.xulo → b.xulo → a.xulo`. Detection is specified in
[visibility.md](visibility.md); the reason is the same: a cycle has no
topological initialization order.

## Type-only imports and erasure

A type-only import creates no runtime dependency: the importing module is
checked against the target's declarations, and nothing about the target's
initialization or values is required for the importing module to run. The
binding exists for the checker and is erased everywhere else.

- The name MAY be used in type positions: annotations, type arguments,
  bounds, and return types.
- The name MAY be used as the base of an enum-variant path — in
  `Kind::Admin`, `Kind` resolves as a type name, not as a value, so a
  type-only import of an enum still admits its variant paths.
- The name MUST NOT be used where a value is expected — as an expression, a
  callee, an initializer, or an argument. That is an error, not a runtime
  failure, because no binding exists to be read.

An ordinary (non-`type`) import of a name that is used only in type positions
is legal as well; it simply creates a value binding that the program happens
not to read.

## Name collisions

- Two imports that bind the same local name are an error, whether they come
  from one specifier or from two:

  ```xulo
  import { add } from "math"
  import { plus as add } from "util"   // error: `add` is bound twice
  ```

- An import binding and a file-scope declaration of the same name are two
  declarations in one scope, so that too is a duplicate-binding error.
- An import MAY be shadowed by a declaration in an **inner** scope — a
  function body or block may declare `let math = …` and hide the namespace for
  the rest of that block. Shadowing requires a nested scope exactly as in
  [`../names.md`](../names.md); at file scope it would be a duplicate.
- `_` MAY NOT be used as an import entry or as an alias: it is not an
  identifier ([`../names.md`](../names.md)).

## Visibility of imported names

Importing does not grant re-export. A name that a module imports is part of
that module's *own* view of the world, not of its interface: a module that
imports `add` and does not `pub use { add }` does not export `add`, and its
own importers cannot reach `add` through it. Only `pub` declarations and
`pub use` add names to an interface, and only names that are themselves
exported may be re-exported. The full rules, including how a private name is
protected through every route, are in [visibility.md](visibility.md).

## Well-formedness table

| Violation | Code | Example |
|-----------|------|---------|
| Import names a member the target does not have | `E0601` | `import { nope } from "math"` |
| Import names a member that is not `pub` | `E0602` | `import { helper } from "math"` with `fn helper` unmarked |
| `pub use { x }` where `x` is a private local declaration | `E0602` | `struct S { … }` then `pub use { S }` |
| `pub use { x } from "m"` where `m` has no such export | `E0601` | `pub use { nope } from "math"` |
| `pub use { x } from "m"` where `m` keeps `x` private | `E0602` | `pub use { helper } from "math"` |
| Namespace member is not exported | `E0602` | `math.helper()` where `helper` is private |
| Two bindings for one local name | `E0104` | `import { add } from "a"` then `import { add } from "b"` |
| Import or re-export cycle | `E0603` | `a` imports `b`, `b` imports `a`; self-import |
| `import type` binding used in a value position | `E0606` | `let u = User` after `import type { User }` |
| Specifier unresolvable or ambiguous | `E0607` | `import "nope"` |
| `import` outside the file header | `E0605` | an `import` after a top-level `fn` |
| `pub use { x }` where `x` is not bound here | `E0101` | `pub use { x }` with no declaration or import of `x` |

Codes are stable identifiers; the full scheme is in
[`../type-system/errors.md`](../type-system/errors.md).
