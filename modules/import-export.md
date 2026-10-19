# Import and Export

Names cross module boundaries through three mechanisms, each with one job:
`mod` declarations decide which files *are* modules of this package and how
they nest, `use` moves names of other modules into this file's scope, and
`pub` — on a declaration, on a module, or on a `use` — puts a name into this
module's interface. This chapter defines the forms, what each one binds, how
a path becomes a file, how a package head is named and when packages
initialize, and the errors that follow from breaking these rules.

## Module declarations

A package is a tree of modules, grown from its root by `mod` declarations
([README](README.md)). There are two kinds.

### File-backed modules

```xulo
mod form;
mod internal;
```

A declaration without a body claims a file: `mod form;` in a module whose
*home directory* is `H` resolves to the file `H/form.xulo`. The extension
`.xulo` is implied and MUST NOT be written.

Every module has a home directory:

- The **root module**'s home is the directory of the file the environment
  designates as the package root.
- A file-backed module found at `D/S.xulo` has home `D/S/` — the directory
  named after the file's stem.
- An **inline** module named `m` inside a module with home `H` has home
  `H/m`.

```text
project/
├── main.xulo          root module; home = project/
├── form.xulo          mod form;     → project/form.xulo
├── internal.xulo      mod internal; → project/internal.xulo
└── form/
    └── fields.xulo     mod fields;   → project/form/fields.xulo
```

- A declaration MUST resolve to exactly one file; there is no second
  candidate — no `mod.rs`, no `index.xulo`, no directory module. A file that
  exists but that no `mod` declaration claims is not a module of the
  package: it is not checked, not initialized, and not nameable.
- A file MAY be the body of at most one module declaration, and a
  declaration MUST NOT resolve to the file that contains it. Either
  violation is `E0610`.
- Lookup never leaves the package: homes are built from names, never from
  `..`, so the tree is contained by construction.

The environment designates the root file; how it does so is tooling
([README §1.1](../README.md)). The root module declares `main`
([source-files.md](source-files.md)).

### Inline modules

```xulo
mod tests {
  use form::validate;
  fn rejects_empty() { … }
}
```

An inline module is a block inside its parent's file. Its body is a module
body like any other — its own header, then its own declarations — and its
home directory is `H/m`, so a file-backed child declared inside it still
lands in the tree (`mod extra;` inside `mod tests` in a root file resolves
to `tests/extra.xulo`).

### `pub mod`

`pub` on either form puts the module's *namespace* in the interface: a `pub
mod` name is reachable by other modules through this one, and its `pub`
members are reached through the path. A `mod` without `pub` binds its name
in this file's scope like any other file-scope name and adds nothing to the
interface:

```xulo
pub mod form;          // others may write `form::…` through this module
mod internal;          // nameable here only
```

Module names obey the collision rules below like every other binding.

## Paths

One syntax names everything qualified — module chains, package heads,
variants, associated names:

```text
PathHead    = PackageName | Identifier | "self" | "super"
Path        = PathHead { "::" Identifier }
PackageName = Identifier { "-" Identifier }
```

- **Bare paths are package-root-absolute.** `form::validate` starts at this
  package's root; there is no `crate` prefix and no per-file package
  binding. `self::cache` is the current module; `super::fields` is the
  parent module (above the root, `super` resolves to nothing — `E0607`).
- **A path head is a package name when the manifest says so.** The first
  segment resolves as a dependency of this package first, then as a root
  module of this package; a spelling that would be both is `E0104`
  ([Name collisions](#name-collisions)).
- Paths MAY be used in expressions and types without any preceding `use`
  ([`../expressions/path-and-access.md`](../expressions/path-and-access.md));
  `use` exists to bind a shorter name, not to grant access.

**Hyphen rule.** A package segment's hyphens are written without
surrounding whitespace, and in an expression position the chain is read as a
path only when `::` follows it — so `lsp-server::run()` is a qualified call
while `total - Math::PI` is subtraction.

### Package names and the manifest

A package head resolves through a mapping the environment supplies — the
project's dependency list, as the ecosystem already writes it:

```toml
[dependencies]
lsp-server = "0.8"
clap       = { version = "4.6.6", alias = "clap_lib" }
"@xulo/ui" = { version = "1.2",    alias = "ui" }
```

What this specification requires is only the shape of the rule, as it was
for string specifiers: a package head MUST resolve to exactly one package,
resolution MUST NOT depend on which file asks, and the mapping MUST be
given to the compiler outside the source text ([README §1.1](../README.md)).
Additionally, a conforming manifest maps every dependency to exactly one
**path name**:

| Manifest key | Path name | `alias` |
|--------------|-----------|---------|
| identifier (`clap`) | the key, unless `alias` renames it | optional |
| kebab-case (`lsp-server`) | the key, unless `alias` renames it | optional |
| anything else (`"@xulo/ui"`) | the `alias` | **required** |

When `alias` is present it **replaces** the key as the dependency's only
name in source: the key remains the manifest's identity for tooling, and a
path that spells a replaced key — `clap::args()` under `alias =
"clap_lib"` — is `E0607`. Two dependencies that derive the same path name
are a manifest error, not two ways to reach one package. Every dependency
MUST have a path-writable name — including one never named in source: the
environment initializes it regardless, and its path name is what any file
that later reaches it writes. The manifest format itself —
TOML shape, versions, sources — is tooling ([README §1.1](../README.md));
the compiler consumes the mapping.

## Use forms

All header forms — `mod`, `use`, and `pub use` — appear in the file's
header, before any other top-level declaration, in any order among
themselves ([source-files.md](source-files.md)). A header form written in
the body is `E0605`.

### Single and aliased use

```xulo
use form::validate;
use lsp-server::router::Router as Router3;
```

A `use` binds the last segment as an ordinary file-scope binding holding
exactly what the target exports under that name — a function keeps its
signature, a type keeps its declaration, a constant keeps its type. `as`
renames the local binding only: the right-hand side is a fresh local name
and never affects the target.

The one-segment form (`use lsp-server as ls;`) binds the namespace itself
under an alias; a path that stops at its head MUST carry `as`, since the
head is either already nameable (a root module) or not spellable as a
binding (a kebab package).

Every segment the path walks MUST exist and every name it binds MUST be an
export of the target: an unknown name is `E0601`, a private one is `E0602`.

### Grouped use

```xulo
use ui::{HStack, Button};
use form::{validate, fields::Field};
```

A group takes a path prefix and a brace list of entries relative to it; `as`
renames an entry. The prefix is required — every entry extends a path, and
the forms that do not are the single path and the glob.

### Glob use

```xulo
use lsp-server::*;
```

A glob binds every export of the target at the declaration site. The glob
participates in collisions eagerly: any overlap with another binding, a
`mod` name, or a file-scope declaration is `E0104` at the glob itself, not
at some later use. A glob in which no contributed name is used is `W0102`
once, at the glob.

### Type-only use

```xulo
use type ui::User;
```

`use type` — the marker follows `use`, before the path — binds the listed
path for **type positions only**: annotations, type arguments, trait bounds,
return types, and the base of an enum-variant path. It creates no value
binding and no initialization dependency (see
[Type-only use and erasure](#type-only-use-and-erasure)). Using a
type-only binding where a value is expected is `E0606`. The marker applies
to the whole statement; there is no per-entry form.

### What use does not do

Using does not grant re-export, does not order packages, and does not
change what the target exports. A name bound by `use` is part of
this module's *view* of the world ([Visibility of used
names](#visibility-of-used-names)).

## Export forms

Two families of forms put names into a module's interface.

**Declarations marked `pub`.** Every declaration form carries its own `pub`,
and so does a module:

```xulo
pub fn add(a: int, b: int): int { a + b }
pub struct User { pub name: string, age: int }
pub enum Status { Active, Inactive }
pub trait Shape { fn area(self): float }
pub type Score = int
pub const PI = 3.14
pub let appName = "xulo"
pub mod form;             // the namespace and its pub members
```

**`pub use`.** A re-export adds names to the interface without repeating
`pub` on their declarations, and — unlike a bare `use` — it is one line
from path to interface:

```xulo
pub use ui::hstack::HStack;      // binds HStack here AND exports it
pub use ui::hstack::Button as Btn;
pub use ui::{VStack, Text};
pub use lsp-server::*;           // binds and exports the whole export set
pub use { add, PI };             // re-exports names already in this file
```

- The **path form** follows the `use` target grammar. It reads the name from
  the path, binds it locally (last segment or `as` alias), and adds the same
  name to the interface. The target's name MUST be an export of what the
  path reaches: unknown is `E0601`, private is `E0602`.
- The **local form** `pub use { a, b }` names bindings of *this* module and
  creates none: each name MUST already be part of the interface — a `pub`
  declaration here, a `pub mod` name, or a name brought in by a `use` from
  an export. Re-exporting a private local declaration is `E0602`; naming a
  binding that does not exist is `E0101`. `as` is not written: the name is
  the export.

The complete mapping from declaration to exported name:

| Declaration | Exported name | What clients receive |
|-------------|---------------|------------------------|
| `pub fn f(…)` | `f` | a function of the declared signature |
| `pub struct S { … }` | `S` | the type, its constructor form, and its `pub` members |
| `pub enum E { … }` | `E` | the type, its variants `E::V`, and its `pub` members |
| `pub trait T { … }` | `T` | the trait, to be implemented locally |
| `pub type A = …` | `A` | the transparent alias |
| `pub let x` / `pub let mut x` | `x` | the binding and its declared type |
| `pub const K` | `K` | the constant and its declared type |
| `pub mod m;` | `m` | the namespace: its `pub` members, reached `m::…` |
| `pub use path::x` | `x` | the same name as exported by the path's source |
| `pub use path::x as y` | `y` | the same, bound and exported as `y` |
| `pub use { a, b }` | `a`, `b` | the interface names of this module |
| `impl …` | — | never exported; an `impl` has no name |
| `mod m` (no `pub`) | — | nothing: private modules are not in the interface |

A declaration without `pub` exports nothing, and `impl` blocks are never
part of an interface: importing a type or a trait does not import any `impl`
([`../types/traits.md`](../types/traits.md)).

## Initialization

Ordering *within* a package is the tree's alone — children before parents,
siblings in declaration order — and ordering *between* packages is the one
thing a file cannot state: it belongs to the environment, which MUST
initialize a package's declared dependencies before the package's own tree
begins. Source names packages through paths and declares no edges. The
order — and the `E0611` limit it makes checkable — are specified in
[visibility.md](visibility.md); the manifest mapping above gives the head
its package.

## Type-only use and erasure

A type-only binding creates no runtime dependency: the using module is
checked against the target's declarations, and nothing about the target's
initialization or values is required for the using module to run. The
binding exists for the checker and is erased everywhere else.

- The name MAY be used in type positions: annotations, type arguments,
  bounds, and return types.
- The name MAY be used as the base of an enum-variant path — in
  `Kind::Admin`, `Kind` resolves as a type name, not as a value.
- The name MUST NOT be used where a value is expected — as an expression, a
  callee, an initializer, or an argument. That is `E0606`, not a runtime
  failure, because no binding exists to be read.

**Propagation.** Type-only-ness follows a name through re-exports: a
type-only binding that reaches the interface through `pub use { … }` is
exported type-only, and a client's binding from that entry is type-only in
turn, to the end of any chain of façades. The checker follows every link —
it checks the whole program, not one file's emit — so no marker exists at
the re-export: `use type` is written where the restriction is chosen, and
everything downstream inherits it.

A plain (non-`type`) `use` of a name used only in type positions is legal
too; it simply creates a value binding even though the program needs only
the name. Conversely, a path-based `pub use` of a name its source exports
only as a type (a trait, a type alias) arrives as a type: using it as a
value is the ordinary `E0101` ("names a type where a value is required"),
because the binding was never declared type-only.

## Name collisions

Every collision is decided where the names meet in the header:

- Two `use` bindings of one local name are `E0104`, whether they come from
  one statement or two:

  ```xulo
  use math::add;
  use util::add as add;    // error: `add` is bound twice
  ```

- A `use` binding and a file-scope declaration of the same name are two
  declarations in one scope — `E0104`, same as two declarations of any
  kind.
- A `mod` name is a binding like any other: two `mod` declarations of one
  name, a `mod` against a `use`, all `E0104`.
- A glob overlaps a binding, a `mod` name, or a file-scope declaration —
  `E0104`, reported at the glob.
- A root-level `mod` name or `use … as` alias that equals a manifest path
  name is `E0104`; the hint offers both repairs: rename the local `mod`, or
  give the dependency an `alias` in the manifest. There is no precedence
  rule between "package" and "local": the declaration-site error removes the
  ambiguity from the language rather than resolving it at each use.
- A `use` MAY be shadowed by a declaration in an **inner** scope — a
  function body may declare `let form = …` and hide the binding for the rest
  of that block. Shadowing requires a nested scope exactly as in
  [`../names.md`](../names.md); at file scope it is a duplicate.
- `_` MAY NOT be used as a use entry or an alias: it is not an identifier
  ([`../names.md`](../names.md)).

## Visibility of used names

Using does not re-export. A name a module binds is part of that module's
*view* of the world, not of its interface: a module that binds `add` and
does not `pub use` it does not export `add`, and its own clients cannot
reach `add` through it. Only `pub` declarations, `pub mod`, and `pub use`
add names to an interface, and only names that are themselves exported may
be re-exported. The full rules, including how a private name is protected
through every route, are in [visibility.md](visibility.md).

## Unused names

A `use` entry that this module never uses is reported as `W0102`, at the
statement that binds it. A warning never rejects a program
([`../type-system/errors.md`](../type-system/errors.md)).

- A binding is **used** when it is read anywhere in the module — in a value
  position (called, passed, read through `::` or `.`) or in a type position
  (an annotation, a type argument, a bound, a return type, or the base of an
  enum-variant path). A type-only binding used only in annotations is used.
- A binding re-exported by `pub use` is used: the re-export puts the name in
  the interface, for the same reason a `pub` declaration is never reported
  as unused.
- A glob with no contributed name in use warns once at the glob.
- Shadowing a binding in an inner scope does not use it: if only the
  shadowing declaration is read, the `use` entry is the one that goes
  unused.

```xulo
use math::{add, sqrt};    // `add` is never used: W0102

fn norm(x: float, y: float): float {
  sqrt(x * x + y * y)
}
```

## Well-formedness table

| Violation | Code | Example |
|-----------|------|---------|
| Path names a member the target does not have | `E0601` | `use math::nope` |
| Path names a member that is not `pub` | `E0602` | `use math::helper` with `fn helper` unmarked |
| `pub use { x }` where `x` is a private local declaration | `E0602` | `struct S { … }` then `pub use { S }` |
| `pub use path::x` where the path's source keeps `x` private | `E0602` | `pub use math::helper` |
| Type-only binding used in a value position | `E0606` | `let u = User` after `use type ui::User` |
| Path head unresolvable — unknown package, missing `mod` file, `super` above the root | `E0607` | `mod shapes;` with no `shapes.xulo` |
| Header form written in the body | `E0605` | a `use` after a top-level `fn` |
| A `mod` file claimed twice, or by the file that declares it | `E0610` | root `form.xulo` declaring `mod form;` |
| Two bindings for one local name — `use` × `use`, `use` × declaration, `mod` × `mod`, glob overlap, manifest head × local name | `E0104` | `use math::add;` then `use util::add` |
| `pub use { x }` where `x` is not bound in this module | `E0101` | `pub use { x }` with no declaration or use of `x` |
| A path that spells a manifest key an `alias` replaced | `E0607` | `clap::args()` with `alias = "clap_lib"` |
| A `use` entry never used in this module | `W0102` | `use math::add` with `add` never referenced |

Codes are stable identifiers; the full scheme is in
[`../type-system/errors.md`](../type-system/errors.md).
