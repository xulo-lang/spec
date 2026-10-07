# Visibility

Every name a Xulo program declares has exactly one visibility: it is either
private to the file that declares it, or it is `pub` and may be reached from
other files. This chapter defines the default, where `pub` may be written, what
it opens, how visibility shapes package boundaries and import cycles, and the
diagnostics that follow from breaking these rules. The forms that move names
across files are specified in [import-export.md](import-export.md).

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
module. A named import, a namespace member, a re-export, and a direct qualified
reference all fail on the same ground, and there is no spelling that bypasses
the check. Members are private by default too (see below), so even an exported
type may keep its fields and methods to itself.

## `pub` on declarations

`pub` MAY be written on `fn`, `struct`, `enum`, `trait`, `type`, `let`,
`const`, and on `use` (`pub use`, specified in
[import-export.md](import-export.md)). On each of these it exports exactly the
one name the declaration introduces, so that another module which imports it
may refer to it:

```xulo
pub fn add(a: int, b: int): int { a + b }
pub struct User { name: string, age: int }
pub enum Status { Active, Inactive }
pub trait Shape { fn area(self): float }
pub type Score = int
pub const PI = 3.14
pub let appName = "xulo"
```

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
  help** — a module that cannot import `User` cannot reach `User.name` either,
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

A package is the set of modules reachable from one entry file
([README](README.md)). Visibility decides what each link of that reachability
chain may see:

- **Inside one module**, every declaration is mutually visible: functions,
  types, constants, and members refer to each other regardless of `pub`, order,
  or scope kind.
- **Across modules**, only `pub` names cross. An import that names a private
  member is an error, and a member access on an imported value reaches only the
  `pub` members of its type.
- **An imported `pub` name is visible to the importing module** for the rest of
  that file, exactly as if it had been declared there — but it is part of the
  importer's *view*, not of its interface: importing does not re-export
  ([import-export.md](import-export.md)).

```xulo
// math.xulo
pub fn add(a: int, b: int): int { a + b }
fn helper(n: int): int { n }

// main.xulo
import { add, helper } from "math"   // error: `helper` is not public
```

A package boundary is therefore nothing more than a file boundary crossed by an
import, with `pub` as its gate; the tooling that lays packages out is outside
this specification ([README §1.1](../README.md)).

## Cycles

The import graph has one edge for each `import` declaration and one for each
`pub use { … } from "…"` re-export. A direct cycle (a module that imports
itself) or a transitive cycle (a module reachable from itself through any chain
of edges) is a **compile-time error**:

```xulo
// a.xulo
import { b } from "./b"

// b.xulo
import { a } from "./a"    // error[E0603]: import cycle
```

Detection is a traversal of the import graph: whenever a module is revisited
along the path that started at it, the graph contains a cycle. The diagnostic
MUST name the cycle path in order, for example
`a.xulo -> b.xulo -> a.xulo`, so that the offending files can be found without
reconstructing the graph by hand. The reason cycles are rejected is that a
cycle has no topological order: neither initialization nor visibility can be
resolved module by module. The diagnostics are specified in
[`../type-system/errors.md`](../type-system/errors.md).

## Shadowing vs visibility

Shadowing and visibility answer different questions: shadowing decides *which
declaration of a name* a use refers to inside one file; visibility decides
*whether a file may see a declaration at all*.

- A private name MAY be shadowed by an inner declaration exactly as any other
  name — a local `let count = 0` hides a module-level `count` for the rest of
  its block ([`../names.md`](../names.md)).
- An imported name MAY be shadowed by a binding in an inner scope: a function
  body or block may declare `let math = …` and hide the imported namespace for
  the rest of that block. At file scope this is not shadowing but a duplicate
  binding, which is an error.
- Shadowing never grants visibility. Hiding a name locally says nothing about
  the declaring module's private names, and importing one name never makes the
  rest of the target module's declarations reachable.

## Private types in public interfaces

A `pub` declaration MUST NOT expose a non-`pub` type: every type written in the
signature of an exported declaration — a parameter type, a return type, the
type of a `pub` field, of a `pub let`, or of a `pub const`, the right-hand side
of a `pub type`, and a bound of a `pub` generic declaration — MUST be visible
outside the module that declares it.

```xulo
// cache.xulo
struct Cache { entries: int }              // private type

pub fn load(): Cache { Cache(entries: 0) } // error[E0604]: `Cache` is not public
```

Without this rule an importer could obtain values it can neither name nor
manipulate, and every call through the interface would smuggle a private type
across the boundary. Types that are visible everywhere — the built-in type
names, the prelude, and types the same module exports — satisfy the rule
without any modifier. The diagnostic, with its exact message shape, is
specified in [`../type-system/errors.md`](../type-system/errors.md).

## Summary table

| Declaration form | Default visibility | Effect of `pub` |
|------------------|--------------------|-----------------|
| `fn` at file scope | module-private | the function is importable |
| `struct`, `enum`, `trait`, `type` | module-private | the type is importable |
| `let`, `const` at file scope | module-private | the binding and its declared type are importable |
| Struct field | module-private | readable and writable from modules that see the struct |
| Method in an `impl` | module-private | callable from modules that see the type |
| Enum variant | follows its enum | no modifier exists; variants ride on the enum's visibility |
| Trait method | follows its trait | no modifier exists; members ride on the trait's visibility |
| `impl` block | no visibility | never exported; `impl` has no name |
| Nested `fn` | block-local | `pub` does not apply |
| Import binding | this module's view | not part of the interface unless re-exported with `pub use` |
| `pub use` | already public | adds the named bindings to this module's interface |
