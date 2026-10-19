Every name a Xulo program declares has exactly one visibility: it is either
private to the module that declares it, or it is `pub` and may be reached
from other modules. This chapter defines the default, where `pub` may be
written, what it opens, how visibility shapes package boundaries and
initialization, and the diagnostics that follow from breaking these rules.
The forms that move names across modules are specified in
[import-export.md](import-export.md).

## Default private

Every declaration is module-private unless it is marked `pub`. A private name
may be used anywhere inside the file that declares it — from its declaration to
the end of the file, and, for the declarations that forward-reference
([source-files.md](source-files.md)), throughout the whole file — and nowhere
else.

```xulo
struct Account { balance: int }        // private type
fn debit(a: Account, n: int): Account { a }   // private function

pub struct Ledger { entries: int }     // exported type
```

Privacy attaches to the *name*: no route reaches a private name from another
module. A `use` binding, a path segment, a re-export, and a direct qualified
reference all fail on the same ground, and there is no spelling that bypasses
the check. Members are private by default too (see below), so even an exported
type may keep its fields and methods to itself.

## `pub` on declarations

`pub` MAY be written on `fn`, `struct`, `enum`, `trait`, `type`, `let`,
`const`, on `mod`, and on `use` (`pub use`, specified in
[import-export.md](import-export.md)). On each of these it exports exactly
the one name the declaration introduces, so that another module which uses
it may refer to it:

```xulo
pub fn add(a: int, b: int): int { a + b }
pub struct User { name: string, age: int }
pub enum Status { Active, Inactive }
pub trait Shape { fn area(self): float }
pub type Score = int
pub const PI = 3.14
pub let appName = "xulo"
pub mod form;            // the namespace, and through it the `pub` members
```

A `mod` without `pub` binds its name in the declaring module's scope only;
`pub mod` puts the namespace in the interface, so other modules may path
through it (`form::validate`) exactly as they path through a dependency's
root.

An `impl` block carries no visibility of its own: `impl` introduces no name and
is never part of an interface, so there is no `pub impl`. The visibility of the
methods inside an `impl` is decided by each method's own `pub`:

```xulo
struct Counter { n: int }

impl Counter {
  pub fn value(self): int { self.n }          // callable from other modules
  fn bump(mut self) { self.n = self.n + 1 }   // private to this module
}
```

## `pub` on members

Struct fields and methods are private by default; `pub` opens a single member
to modules that can already see the type that declares it.

```xulo
pub struct User {
  pub name: string   // readable and writable outside this module
  age: int           // reachable only inside this module
}
```

- The core rule: **if the type itself is not `pub`, its `pub` members do not
  help** — a module that cannot name `User` cannot reach `User.name` either,
  no matter how the field is declared.
- A field's `pub` controls reads and writes alike: assignment through
  `user.name = …` from another module requires the field to be `pub`, and the
  ordinary mutability rules apply on top of visibility.
- **Enum variants are reachable through the enum.** Variants carry no
  visibility modifier of their own: every variant of a `pub` enum may be
  constructed (`Status::Active`) and matched from any module that can see the
  enum, and no variant of a private enum is reachable outside its file.
- **Trait members are public when the trait is `pub`.** The methods a `pub`
  trait declares carry no modifier either: they are visible wherever the trait
  is visible, because another module must both implement them and dispatch
  through them. A private trait exposes nothing, since the trait itself cannot
  be named.

## Package boundaries

A package is the tree of modules reachable from its root by `mod`
declarations ([README](README.md)); visibility decides what each link of
that tree may see:

- **Inside one module**, every declaration is mutually visible: functions,
  types, constants, and members refer to each other regardless of `pub`, order,
  or scope kind.
- **Across modules**, only `pub` names cross. A path that names a private
  member is an error (`E0602`), and a member access on an imported value
  reaches only the `pub` members of its type.
- **A `pub` name brought in by `use` is visible for the rest of that file**,
  exactly as if it had been declared there — but it is part of the using
  module's *view*, not of its interface: using does not re-export
  ([import-export.md](import-export.md)).
- **A dependency's root is a package boundary, not a file one.** Its modules
  are reached only through paths the dependency's interface exports; the tree
  inside it is its own.

```xulo
// math.xulo
pub fn add(a: int, b: int): int { a + b }
fn helper(n: int): int { n }

// main.xulo
use math::{add, helper};    // error: `helper` is not public
```

How a path head becomes a package is specified in
[import-export.md](import-export.md); the package registry protocol,
project manifests, and composition tooling are outside this specification
([README §1.1](../README.md)).

## Initialization

Modules initialize in an order fixed by structure, never computed from a
dependency graph:

1. **Packages** initialize in the order the environment provides — a
   package's declared dependencies before the package itself. The source
   states no package edges; an environment that orders a dependent before
   its dependency is a tooling failure outside this specification.
2. **Within a package**, the tree initializes in **post-order**: a module's
   children before the module itself, siblings in the order their `mod`
   declarations appear in the parent's header.
3. **Within a module**, file-scope `let` and `const` initializers run in
   declaration order ([source-files.md](source-files.md)), and every
   module's initializers complete before `main` runs
   ([`../memory-and-runtime.md`](../memory-and-runtime.md)).

`use` and `pub use` create no order edge: they move names, not state. The
tree cannot cycle — each file is the body of at most one `mod` declaration
(`E0610`) — so a structural order always exists, and cycles written between
`use` lines (`a` uses `b`, `b` uses `a`) are legal to keep: nothing computes
an order from them. The former cycle error `E0603` is retired and reserved.

Because the order is fixed and known, one compile-time check guards the
case structure alone cannot: a file-scope initializer MUST NOT directly
read a file-scope `let` of a module that initializes after it, and MUST NOT
directly call a function declared in such a module. Either is `E0611`, and
the check is semantic — a `use`d binding of the same target counts.
Indirection is not chased: an initializer that calls its own function,
which then reads or calls into the later module, is held to the same limit
the within-file forward-reference rule already has
([source-files.md](source-files.md)); the checker builds no initialization
call graph.

```xulo
// root: mod stats; mod config;   — stats initializes first
// stats.xulo
use config::N;             // header: naming the binding is fine
let snap = N               // error[E0611]: the read runs before config initializes
```

Reading the earlier-initialized state of another module — a `const`, a
type, a function *name*, or a `let` of any module the order already
completed, including every dependency package — is sound and allowed.

## Shadowing vs visibility

Shadowing and visibility answer different questions: shadowing decides *which
declaration of a name* a use refers to inside one module; visibility decides
*whether a module may see a declaration at all*.

- A private name MAY be shadowed by an inner declaration exactly as any other
  name — a local `let count = 0` hides a module-level `count` for the rest of
  its block ([`../names.md`](../names.md)).
- A `use`d name MAY be shadowed by a binding in an inner scope: a function
  body may declare `let form = …` and hide the imported namespace for the
  rest of that block. At file scope this is not shadowing but a duplicate
  binding, which is an error (`E0104`).
- Shadowing never grants visibility. Hiding a name locally says nothing about
  the declaring module's private names, and using one name never makes the
  rest of the target module's declarations reachable.

## Private types in public interfaces

A `pub` declaration MUST NOT expose a type that is not part of *this
module's interface*: every type written in the signature of an exported
declaration — a parameter type, a return type, the type of a `pub` field, of
a `pub let`, or of a `pub const`, the right-hand side of a `pub type`, a
bound of a `pub` generic declaration — MUST be either declared `pub` here,
exported here by a `pub use` chain, or universally visible (a built-in type
name, or a name of the prelude).

```xulo
// cache.xulo
struct Cache { entries: int }              // private type

pub fn load(): Cache { Cache(entries: 0) } // error[E0604]: `Cache` is not public
```

```xulo
// façade.xulo
use ui::Widget;
pub fn render(w: Widget) { … }             // error[E0604]: `Widget` is not exported here
pub use ui::Widget;                        // the repair: put it in the interface
```

The second case is the façade leak: a signature that mentions a type the
module can see but does not export would give other modules a value they can
neither name nor manipulate — the interface must contain everything its own
signatures stand on. A dependency's type used in a `pub` signature therefore
travels with an explicit `pub use`, which is exactly the rule Swift's
interface-leak check enforces and the reason `public import` there is not
re-export: seeing a module is not publishing it. The diagnostic, with its
exact message shape, is specified in
[`../type-system/errors.md`](../type-system/errors.md).

## Summary table

| Declaration form | Default visibility | Effect of `pub` |
|------------------|--------------------|-----------------|
| `fn` at file scope | module-private | other modules may `use` it |
| `struct`, `enum`, `trait`, `type` | module-private | other modules may `use` the name |
| `let`, `const` at file scope | module-private | other modules may `use` the binding and its declared type |
| `mod` declaration | module-private | the namespace is reachable through this module, `mod::…` |
| Struct field | module-private | readable and writable from modules that see the struct |
| Method in an `impl` | module-private | callable from modules that see the type |
| Enum variant | follows its enum | no modifier exists; variants ride on the enum's visibility |
| Trait method | follows its trait | no modifier exists; members ride on the trait's visibility |
| `impl` block | no visibility | never exported; `impl` has no name |
| Nested `fn` | block-local | `pub` does not apply |
| `use` binding | this module's view | not part of the interface unless re-exported with `pub use` |
| `pub use` | binds and exports | adds the named bindings to this module's interface |
