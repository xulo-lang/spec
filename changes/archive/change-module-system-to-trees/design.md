# Design: Module System as Trees

This document records the alternatives for the module system, the design
chosen, and the consequences that choice carries: what a package is, how a
tree is built, where package names come from, which verb binds which cell,
which operator owns which lane, and what initialization order means once the
import graph is gone. The prose that replaces the chapter text is in
[specs/](specs/); the motivation and the scope are in
[proposal.md](proposal.md).

## The four-level vocabulary

### Project, workspace, package, module — RECOMMENDED

```text
project      the buildable unit: a directory with a manifest
workspace    several projects composed        (tooling vocabulary)
package      a tree of modules, one root, grown by `mod` declarations
module       one file, or one inline block: the unit of checking and visibility
```

- **Pro:** every level is the level some neighboring ecosystem already named
  (Cargo's package/workspace, go.mod's module, npm's package), so a reader
  arriving from any of them lands on the word they expect. The two upper
  levels are recorded but *not specified* — they are exactly the layer
  [README §1.1](../../../README.md) already excludes (registry, toolchain,
  manifests as files), so naming them costs the specification nothing.
- **Con:** four words where two would do; a reader may wonder why "package"
  and "project" differ.

**Decision.** The specification names all four, specifies the bottom two,
and adds one §1.1 row placing projects, workspaces, and the manifest format
in tooling
([specs/README.md](specs/README.md)). Everything normative — membership,
layout, lookup, paths, visibility — speaks only of *packages* and *modules*.

### Three levels, workspace unnamed — rejected

Dropping the word *workspace* would be free, but the word will appear in
every tooling document that surrounds this specification anyway; a vocabulary
note in §1.1 is cheaper than a permanent translation table between this
specification and its ecosystem.

### Package = source directory — rejected

"A package is a directory of sources" makes membership a filesystem
accident: dropping a file into the directory makes it part of the language's
program. Membership must be something the program *says* — see the next
section.

## What a package is

### The tree reachable from a root by `mod` declarations — RECOMMENDED

- **Pro:** membership is declared, not inferred. Every module in the package
  is named by a chain of `mod` declarations from one root, so the package is
  exactly what its author wrote, checked from the root outward. A stray file
  is not a module of anything — it is not checked, not initialized, not
  reachable, and the tooling is free to say so. The tree is the same
  structure the runtime initializes along (see
  [Initialization](#initialization-and-the-retirement-of-e0603)), so
  "package", "tree", and "initialization order" are one concept instead of
  three.
- **Con:** converting an existing loose directory of files into a package
  requires writing the `mod` declarations once.

**Decision.**

> A **package** is the tree of modules reachable from its **root module** by
> following `mod` declarations — file-backed (`mod name;`) and inline
> (`mod name { … }`) alike. The root module is the file the environment
> designates as the package root. A `.xulo` file that no `mod` declaration
> claims is not a module of the package: it is not checked, not initialized,
> and not nameable.

### The set of modules reachable from one entry file by imports — rejected

This is the current wording ([modules/README.md](../../../modules/README.md)),
and it cannot coexist with the plural package of
[import-export.md](../../../modules/import-export.md) ("the program's
packages"): one definition says membership is reachability (a single bag
swallowing every third-party module), the other says the environment names
several packages. Reachability by *use* also makes membership depend on a
`use` line — deleting an import silently removes a module from the package.
Import edges answer "who may see whom"; they are the wrong mechanism for
"what is this program".

### The tree plus every file under the package directory — rejected

Directory-inclusive membership was the middle ground (stray files still
checked), but it resurrects the filesystem as the source of truth and makes
the tooling's layout rules part of the language. Files are in the package
because a declaration claims them.

## Building the tree

### `mod` declarations, Rust-nested file lookup — RECOMMENDED

```xulo
// main.xulo — the package root, designated by the environment
mod form;              // → form.xulo, beside this file
mod internal;          // → internal.xulo
mod tests { … }        // inline: lives inside this file

// form.xulo
mod fields;            // → form/fields.xulo: a file's children live in a
                       //   directory named after the file's stem
mod shape { … }        // inline child of form, inside form.xulo
```

**Lookup rule.** Every module has a *home directory*:

- The root module's home is the directory of the file the environment
  designates.
- A file-backed module found at `D/S.xulo` has home `D/S/` — the directory
  named by its stem.
- An inline module `m` inside a module with home `H` has home `H/m`.

A bodyless `mod c;` in a module with home `H` resolves to the file
`H/c.xulo`; the extension `.xulo` is implied and MUST NOT be written. There
is no second candidate: no `mod.rs`, no `index.xulo`, no directory module —
a declaration names a file, always.

- **Pro:** one rule produces the familiar nesting (`mod fields;` inside
  `form.xulo` opens `form/fields.xulo`) while keeping the current
  specification's promise that a module is *a file, never a directory*.
  Structure is written in the files themselves, so the tree is readable
  without a directory listing and greppable without walking one. Names are
  `Identifier`s, so paths compose from identifiers only — no case folding,
  no kebab-to-camel guessing at lookup time.
- **Con:** files accumulate header lines as a codebase grows; a file can no
  longer be dropped into a directory and picked up implicitly.

**Decision.** Declaration-driven membership with the home-directory lookup
above. A file may be the body of at most one module declaration, and a
declaration MUST NOT resolve to the file that contains it; either violation
is `E0610`. The full text is in
[specs/import-export.md](specs/import-export.md).

### Every file is a module, `mod` only for grouping — rejected

Keeping file-as-module alongside declarations gives two ways to become a
module — an implicit one and an explicit one — and the implicit one is the
one that makes "reachability" and "membership" drift apart again. The tree
needs a single membership mechanism; Rust, Swift, and Go all chose
declaration-driven trees for the same reason. `mod` is either how you join
the tree or it is decoration.

### Directory modules (`mod.rs`, `index.xulo`) — rejected

An index file is a second declaration location whose contents *are* the
directory's module: two places (the index and the stem file) can claim the
same child set, and "what is `lib`?" becomes "which file does this toolchain
prefer?". The lookup rule above has exactly one candidate per name, which is
what makes `E0610` a checkable invariant instead of a configuration quirk.

### One file, one module, no inline blocks — rejected for this design

Inline `mod` was the one structural extension discussed beyond Rust: it
keeps closely-related helpers (tests, derivations) with their parent without
a file each, and it makes "several modules per file" legal without making
it common. Forbidding it would save a grammar production but would push
every grouping instinct back into `pub use` façade files.

## No `crate`

`crate` does three jobs in Rust; each has a cheaper answer in Xulo:

| Rust duty | Rust spelling | Xulo answer |
|-----------|---------------|-------------|
| Absolute path from package root | `crate::a::b` | bare paths *are* root-absolute: `a::b` |
| Disambiguate package names from local names | `crate` vs extern crate name | the manifest gives every dependency one path name; heads never collide with `mod` names (`E0104` at declaration) |
| Intermediate visibility tier | `pub(crate)` | out of scope: two levels only, `pub` and private ([README §1.1](../../../README.md) row for package tiers is not added — visibility crosses modules, not packages) |

- **Pro:** no keyword, no new tier, no prefix in the common case; the reader
  of `lsp-server::router::Router` sees the whole road with nothing standing
  in for "here is the package boundary" — the boundary is the head of the
  path.
- **Con:** a Rust reader must unlearn `crate::`; a local module whose name
  equals a package name would be ambiguous, which is why the collision is an
  error rather than a precedence rule.

**Decision.** `crate` is not a keyword and not reserved. Bare paths default
to package-root-absolute; `self` and `super` give the relative forms.
The alternative spellings were rejected:

- **Keep a `crate` keyword** — a word that prefixes nothing, on a language
  whose paths already start at the root; pure borrowing.
- **`extern crate` / per-file dependency declarations** — dependencies are a
  property of the package (its manifest), not of a file; per-file
  redeclaration splits one truth into N headers.
- **`pub(crate)` and other package tiers** — a third visibility level for a
  boundary the language otherwise never mentions; if a future change needs
  it, it arrives as its own change with its own diagnostics.

## Package names and the manifest

### One path name per dependency, provided by the manifest — RECOMMENDED

The manifest of a project lists its dependencies; from the compiler's point
of view that list is only a **mapping from real package names to path
names**:

```toml
[dependencies]
lsp-server = "0.8"
clap       = { version = "4.6.6", alias = "clap_lib" }
"@xulo/ui" = { version = "1.2",    alias = "ui" }
```

Derivation rules:

| Manifest key | Path name | `alias` |
|--------------|-----------|---------|
| identifier (`lsp-server`) | the key | optional (renames) |
| kebab-case (`lsp-server`) | the key | optional |
| anything else (`"@xulo/ui"`, quoted keys) | — | **required**: paths can only spell what the lexer accepts |

- **Pro:** strings leave the language entirely; names are chosen once, at
  the one place where humans already negotiate them; two keys that derive
  the same path name are a manifest error, not two ways to reach one module.
  The environment-orientation of the current text ("the mapping is determined
  outside this specification") survives intact — the specification still
  defines only *the shape of the rule*: a path head resolves to exactly one
  package, resolution does not depend on the asking file, and the mapping is
  supplied by the environment ([specs/import-export.md](specs/import-export.md)).
- **Con:** the spec must state a mapping contract rather than nothing; the
  manifest format itself stays in tooling (§1.1), so a second document reads
  alongside this one.

**Decision.** The specification fixes the shape of the mapping — unique,
unambiguous, file-independent — plus the derivation table above as the
default any conforming manifest format provides. When `alias` is present it
*replaces* the key: exactly one name reaches the package from source, the
key remains manifest-internal, and a path that spells the key is `E0607`.
TOML, versions, sources, and registry protocol remain §1.1 material
([specs/README.md](specs/README.md)); TOML is quoted above only because it
is the ecosystem's format and the shape must be illustrated by something.
Every declared dependency MUST have a path-writable name (an `alias` where
the key is not writable), including dependencies never named in source —
the environment initializes them, and the path name is what any file that
later reaches them would write.

### String specifiers in source — rejected

Two lexical shapes for one job (a string in `import`, an identifier path in
`use`) is the split this change exists to close; strings also cannot be
checked by name resolution (no `E0101`-style reachability, no completion),
and they made `E0607` a string-literal diagnostic. The specifier's remaining
duty — "exactly one target, independent of the asking file" — is inherited
by the path rule unchanged.

### Environment-mapped names with no manifest — rejected

Mapping names only "in the environment" (the current E0607 text) leaves the
source without any record of where a head comes from: `math` in one file and
`@xulo/ui` in another would be unverifiable by anything the reader can open.
The manifest is where that record already lives; the spec needs only the
shape of what it feeds the compiler.

## Paths

### One path syntax, `::`, root-absolute by default — RECOMMENDED

```text
PathHead   = PackageName | Identifier | "self" | "super"
Path       = PathHead { "::" Identifier }
PackageName = Identifier { "-" Identifier }
```

- `form::validate` — root-absolute: `form` is a root module of this package.
- `self::cache` — the current module; `super::fields` — its parent (`super`
  moves from the reserved list into the keyword table; above the root is
  unresolvable, `E0607`).
- `lsp-server::router::Router` — a dependency, named by its manifest path
  name; the head is a package name only where the manifest says so.
- The same path writes module chains, variants (`Status::Active`), and
  associated names (`Math::PI`, `Area::area`) — one syntax, one reader.

**Hyphen rule.** A package segment is written with no whitespace around its
hyphens, and in an expression position the hyphen chain is read as part of a
path only when a `::` follows it:

```xulo
use lsp-server::router::Router;     // path: the chain is glued and `::` follows
let d = total - Math::PI;           // subtraction: a space separates `total`
let r = lsp-server::run();          // path head: glued chain followed by `::`
let e = a - b::c;                   // subtraction: `a` is not glued to `-`
```

Without the `::`-lookahead, `total - Math::PI` would be ambiguous; with it,
the ambiguity is decided by two facts a reader can see (glue and what
follows). The alternative — never allowing kebab names in paths, aliasing
them in the manifest always — was rejected: it makes `use lsp-server::*`
impossible to write, which is the natural spelling the ecosystem already
uses.

- **Pro:** one grammar for every qualified name; `.` is left to values
  ([Operator lanes](#operator-lanes)); root-absolute bare paths remove
  `crate` without removing a way to say "from the top".
- **Con:** `self`/`super` become keywords (two words off the identifier
  pool); the hyphen rule is a real lexical corner, stated once.

**Decision.** As above; full text in
[specs/import-export.md](specs/import-export.md) and
[specs/grammar.md](specs/grammar.md).

### Relative specifiers (`./`, `../`) — rejected

Directory-relative spelling answers the same question as `self`/`super`
with a second syntax that only *imports* have — and paths in expressions
never had it. `./shapes` also reintroduces the string's freedom to escape
the package (`../../etc`), which the home-directory rule makes structurally
impossible: `mod` paths only ever append a name to a home that is already
inside the package.

### Path specifiers with a `from` clause (`use x from "m"`) — rejected

A fourth dialect: ES grammar under a Rust verb. Every form the verb system
already covers is spelled without it.

## Verb system

### The four cells in other languages

| | binds locally | binds nothing |
|---|---|---|
| **private use** | Rust `use`, Swift/Go/Kotlin `import`, Scala `import` | — |
| **bind and export** | Rust `pub use`, Scala `export`, Dart/Swift/`C++` re-export forms | — |
| **export only (forward)** | — | ES/TS/Dart `export … from` |

Two facts from the twelve-language survey (JavaScript, TypeScript, Dart,
Rust, Swift, C++20, Scala, Go, Kotlin, Java, C#, MoonBit):

1. **Ten of twelve spell the import `import`**; only Rust uses `use` and C#
   `using`. Xulo's hybrid today — `import` for one lane, `pub use` for the
   other — exists in no surveyed language, and with the third verb gone
   (below) Xulo lands beside Rust: `use` for private use, `pub use` for
   bind-and-export.
2. **The bindless forward exists only in the ECMAScript lineage**
   (JavaScript, TypeScript, Dart). Every systems language that re-exports
   binds while it does it, and four languages (Go, Kotlin, Java, C#)
   re-export not at all. No surveyed language lets a visibility modifier
   decide whether a form binds.

Swift is the closest precedent for Xulo's export rule: `public import` is
*not* re-export (SE-0409 explicitly reserved the spelling from meaning
re-export, because a library that merely depends on a module must not
publish it), and real re-export is a separate, deliberate act — Xulo's
`pub use` is exactly that deliberate act, and the interface-upper-bound rule
of `E0604` is Swift's interface-leak rule by another name. Dart shows the
other boundary: both `import` and `export` are header-only, which is the
position rule this change adopts for every header form.

### `use`, `pub use` — RECOMMENDED

| | binds locally | binds nothing |
|---|---|---|
| **private use** | `use form::validate;` (single, `{a, b}`, `*`, `as`, `use type`) | — |
| **bind and export** | `pub use ui::hstack::HStack;` — one line, the façade | — |
| **export only** | — | `pub use { HStack };` — re-exports names already in this file's scope |

- **Pro:** the verb does one job per form and the modifier never flips the
  job — two verbs, four cells, and no bindless form in the language at all:
  the cell that cost two declarations today (`import` + `pub use { x }`)
  becomes the line a façade is written in, and the initialization job that
  needed a third verb belongs to the environment, not to source text. The
  forward form retires: the only language family that has it is the family
  whose `export` is a declaration rather than a modifier, and Xulo's `pub`
  is a modifier. `from` and `import` leave the keyword table with it.
- **Con:** readers migrating from ESM lose `export { x } from` and the
  explicit `import "m";` they may look for; a façade that wants *both* a
  local binding and the export writes `pub use` (which does both), not
  `use` plus a re-export.

**Decision.** Two verbs, header-bound:

```xulo
use lsp-server::router::Router;     // bind: Router in this file's scope
use ui::{HStack, Button};           // group: prefix + entries, `as` allowed
use lsp-server::*;                  // glob: every export, declared strictly
use type ui::User;                  // type-only binding (see below)
use lsp-server as ls;               // alias the namespace itself

pub use ui::hstack::HStack;         // bind + export (path form)
pub use { HStack, Button };         // export names already in this file
```

- A `use` target must bind something: a path with at least one segment after
  its head, a group, a glob, or an alias — `use lsp-server;` is not a form
  (it would bind a hyphenated name that is already nameable by path, and a
  package's initialization needs no mention: it follows from the manifest
  and the environment).
- `pub use` path form: exports the last segment or the `as` alias, and binds
  it locally — cell ② in one line.
- `pub use { … }` local form: every name MUST already be in this module's
  interface — a `pub` declaration here, a `pub mod` name, or a name a `use`
  brought in from an export (as today: `E0601`, `E0602`); no new binding is
  created, and `as` is not written (the name it would rename is the export).
- Ordering between packages is the environment's to honor — dependencies
  before dependents — so source declares no package edge and no
  side-effect form exists.

Alternatives rejected:

- **Keep `import … from "m"`** — the string lane stays, two verbs stay, and
  the cells stay split; nothing in the motivation is answered.
- **Keep a side-effect `import pkg;` third verb** — the manifest already
  declares the package and the environment already orders it before its
  dependents, so a source edge restates the build file in the language and
  spends a keyword doing it; a head-only `use pkg;` in its place would be
  the same bindless cell, with a `use` spelling that binds nothing.
- **`use … from` hybrid** — ES grammar with a Rust keyword; a new dialect
  rather than a fix.
- **Bindless re-export kept as a form** — no systems language needs it; with
  path forms, "forward and also bind" is the cheaper line, and a form that
  binds nothing while looking like it does is the exact confusion `E0601` and
  `E0602` were spent on.
- **Position: `pub use` may appear anywhere** (today's rule) — two position
  rules for three forms; Dart shows the alternative: one header for all of
  them. `E0605` covers `mod`, `use`, and `pub use` written in the
  body.

### Type-only — `use type`, propagation, no `pub use type`

**Decision.**

> `use type` creates a binding for **type positions only**: annotations,
> type arguments, bounds, return types, and the base of a variant path. Used
> where a value is required it is `E0606`. Type-only-ness **propagates**:
> when a type-only binding is put in the interface by `pub use { … }`, the
> exported entry is type-only, and a client's binding from that entry is
> type-only in turn, all the way down a chain of façades. There is no
> `pub use type` spelling: the checker sees every link of the chain — it
> checks the whole program, not one file's emit as a TypeScript transpiler
> does — so the marker lives at the only place it is needed, the binding
> that restricts *this file's* view.

What the checker derives for a path-based `pub use` follows the source: a
name the source module exports only as a type (a trait, a type alias)
arrives as a type, and using it as a value is the ordinary "names a type
where a value is required" error (`E0101`) — `E0606` stays the diagnostic
for *bindings that were declared type-only*, which keeps one code, one
message shape.

Alternatives rejected: **no type-only form at all** (a client that only
wants the type pays for an initialization dependency it cannot see);
**`pub use type` marker** (whole-program checking makes it redundant — it
would document what the checker already knows); **type-only as a property of
paths rather than bindings** (then every qualified type annotation would
need erasure rules; the binding is where erasure happens).

## Operator lanes

**Decision.**

> `::` owns the **declaration lane**: paths (`ui::hstack::HStack`), module
> and package heads (`self::cache`, `lsp-server::router`), variants
> (`Status::Active`), associated and namespace names (`Math::PI`,
> `Time::now`, `Task::all`, explicit trait dispatch `Area::area`).
> `.` owns the **value lane**: fields, methods, map member entries, tuple
> positions, and the null-tolerant `?.` chain.

The flip of built-in namespaces and trait dispatch from `.` to `::` is what
makes the lanes absolute: today `Math.max` and `user.name` share a spelling
while naming different *kinds* of thing (a namespace member vs a member of a
value). After the split, a reader can tell from the separator whether they
are walking names (declarations) or reaching into a value — the same
distinction Rust draws with `::` and `.`, and the reason `Theme::Dark` was
already `::`.

- **Pro:** one rule, no exceptions; built-in namespaces stop being a
  privileged case of `.`; explicit dispatch and variant construction reuse
  the path syntax instead of overloading a value operator.
- **Con:** every example in the specification that calls an intrinsic or a
  trait method changes (`Math.abs` → `Math::abs`) — a mechanical sweep,
  enumerated in [tasks.md](tasks.md); a reader coming from JavaScript must
  unlearn `Math.floor`.

Alternatives rejected: **`.` everywhere (Go/JS style, packages as objects)**
— makes namespaces values, which collides with the path model and with
variant syntax (`Theme.Dark` was already rejected in the current chapter);
**`::` everywhere, including fields** — `user::name` on a map entry reads as
a path, and optional chaining `?.` has no `::` spelling; **keep built-ins on
`.`** — the privileged exception is exactly what the lane rule exists to
remove.

## Name collisions and globs

**Decision.** Every collision is a **declaration-site error**: the checker
knows all contributors when the header is read, so nothing waits for a use
site.

- Two `use` bindings of one name — from one statement or two — is `E0104`
  (as today's imports).
- A `use` binding versus a file-scope declaration, a `mod` name, or another
  `mod` name is `E0104`.
- A glob contributes the whole export set of its target at its declaration:
  any overlap with another binding, a `mod` name, or a file-scope
  declaration is `E0104` — reported at the glob, not at some later use.
- A root-level `mod` name or alias that equals a manifest path name is
  `E0104`, with the hint to rename the local `mod` or add an `alias` in the
  manifest — there is no precedence between "package" and "local", because
  the error removes the ambiguity from the language instead of a rule
  resolving it at use time.
- Shadowing in an *inner* scope stays legal, unchanged from
  [`../../names.md`](../../../names.md).

**Alternatives rejected.** **Rust's lazy shadowing** (a later declaration
quietly hides a glob name) keeps programs compiling while changing their
meaning — the glob reads like it brought names in, and it did not; the
strict rule makes `use lsp-server::*` honest: it is a claim that no name in
that package's interface collides here. **Precedence order (local beats
glob beats package)** — a resolution-time rule that hides the collision from
both authors; every ambiguity the precedence would decide is already an
`E0104` with a rename hint.

**Unused names.** `W0102` follows the bindings: an unused single `use`
entry warns at the entry; a glob in which *no* contributed name is used
warns once at the glob; `pub use` never warns (an interface entry is used
by definition).

## Initialization and the retirement of `E0603`

**Decision.** Initialization order is fixed by structure, never computed
from a dependency graph:

1. **Packages** initialize in the order the environment provides — a
   package's declared dependencies before the package itself — and the
   source states no package edges. An environment that orders a dependent
   before its dependency, or that cannot order a set of packages at all, is
   a tooling failure outside this specification.
2. **Within a package**, the tree initializes in **post-order**: a module's
   children before the module itself; siblings in the order their `mod`
   declarations appear in the parent's header.
3. **Within a module**, file-scope `let` and `const` initializers run in
   declaration order (unchanged), and every module's initializers complete
   before `main` runs (unchanged,
   [`../../memory-and-runtime.md`](../../../memory-and-runtime.md)).

`use` and `pub use` create **no** order edge: they move names, not state.
Ordering *between* packages is the one thing a file cannot express, so it
is the one thing source does not try to express: the manifest declares what
belongs to the build, and the environment orders it.

**Why `E0603` retires.** The cycle error existed for one reason: a cycle in
the import graph has no topological initialization order. The new order is
given — the tree is acyclic by construction (each file is the body of at
most one declaration, checked by `E0610`), siblings are ordered by
declaration, packages by the environment — so a graph cycle no longer
prevents an order from existing. Cycles *among `use` lines* (`a` uses `b`,
`b` uses `a`) are still legal to write and now harmless to keep: nothing
computes an order from them. `E0603` is retired and reserved; its number is
never reused.

**Soundness gap — decided: `E0611`.** With edges gone, the old soundness
argument ("everything you can name is initialized before you") now rests on
structure alone, and one case falls outside it: a file-scope initializer of
module `M` that reads a file-scope `let` of module `N` where `N`
initializes *after* `M` observes pre-initialization state. The order is
fixed and known, so what remains is a comparison, not an analysis:

- **Decision — `E0611` for direct reads and direct calls.** A file-scope
  initializer MUST NOT directly read a file-scope `let` of a module that
  initializes after it, and MUST NOT directly call a function declared in
  such a module — the check is semantic, so a `use`d binding of the same
  target counts. One code in the `E06xx` family
  (`` `config::N` is used before `config` initializes ``); the rule lives
  in [specs/visibility.md](specs/visibility.md), the row in
  [specs/errors.md](specs/errors.md). Indirection is not chased — an
  initializer that calls its own function, which then reads or calls into
  the later module, is held to the same limit the within-file
  forward-reference rule already has, and the checker builds no
  initialization call graph. Rust points the same way: push the decision
  to compile time (its statics are const-evaluated, its cycles are a
  compile error) rather than invent a runtime value for a read that should
  not happen.

Alternatives rejected:

- **Reject cross-module reads from initializers entirely** — simpler to
  check (no order comparison), but forbids sound uses (reading a
  dependency's earlier-initialized state) for the sake of the unsound ones.
- **Specify the read as defined pre-initialization state** — impossible to
  give a value in a language without a universal default, and a trap would
  be a runtime failure with no call site to report.

The surrounding rules — the order, `E0603`'s retirement — never depended
on the choice; with `E0611` settled the draft has no open decisions left.

## Entry point

**Decision.** The environment designates the package's root file; the root
module MUST declare `main` with the signatures source-files.md already
allows — `E0608`/`E0609` are unchanged except that "entry module" now means
*the tree root* rather than "the file execution begins from, wherever the
imports led". Every other module of the tree is a library module.

Alternatives rejected: **a magic file name (`main.xulo`)** — layout rules
belong to tooling (§1.1) and a magic name is the directory-module pattern
wearing a hat; **the entry file chosen by imports (today's closure)** —
membership-by-import is the definition this change removes.

## Diagnostics

| Code | Status | Trigger after the change |
|------|--------|--------------------------|
| `E0601` | reworded | a path names a member the target module does not export |
| `E0602` | reworded | a name exists but is not visible here |
| `E0603` | **retired** (reserved) | — (order is structural; never reused) |
| `E0604` | **reworded** | a `pub` signature mentions a type that is not part of *this module's interface* — declared `pub` here or exported by a `pub use` chain (façade leak; Swift's interface-leak rule) |
| `E0605` | **rescoped** | `mod`, `use`, or `pub use` written outside the file header |
| `E0606` | **rescoped** | a type-only binding (`use type`, or propagated through a `pub use` chain) used where a value is required |
| `E0607` | **rescoped** | a path head resolves to nothing — unknown package name, missing `mod` file, `super` above the root |
| `E0608`/`E0609` | unchanged | (entry = tree root) |
| `E0610` | **new** | a `mod` declaration resolves to a file already the body of another module, or to the file that declares it |
| `E0104` | **widened** | any header collision: two `use` bindings, `use` × declaration, `mod` × `mod`, glob overlap, manifest head × local name |
| `E0611` | **new** | a file-scope initializer directly reads a file-scope `let` of a module that initializes after it, or directly calls a function declared in one (indirection is not chased) |
| `W0102` | reworded | an unused `use` entry, or a glob that contributes nothing used |

Message shapes are kept wherever the shape survives the rewording
(`E0601` ``module `math` does not export `nope` ``, `E0602`
`` `helper` is private to `math` ``, `E0604`
``private type `Cache` in public interface``); the full rows are in
[specs/errors.md](specs/errors.md). The `E06xx` family summary row
("modules, visibility, entry point") is unchanged.

## Grammar and keywords

The header becomes one production family; `Declaration` loses `PubUse` and
gains nothing else; expressions gain `PathExpr` in `VariantPath`'s place:

```text
Program      = { HeaderItem } { Declaration } [ Entry ] ;
HeaderItem   = ModDecl | UseDecl | PubUse ;
Declaration  = FnDecl | StructDecl | EnumDecl | TraitDecl | ImplDecl
             | TypeAlias | LetDecl | ConstDecl | TestDecl ;

ModDecl      = [ "pub" ] "mod" Identifier ( ";" | "{" { HeaderItem } { Declaration } "}" ) ;
UseDecl      = "use" [ "type" ] UseTarget ";" ;
PubUse       = "pub" "use" ( UseTarget | "{" [ Ident { "," Ident } [ "," ] ] "}" ) ";" ;

UseTarget    = UsePath | UseGroup | UseGlob ;
UsePath      = PathHead "::" Identifier { "::" Identifier } [ "as" Identifier ] ;
UseGroup     = PathHead "::" "{" [ UseEntry { "," UseEntry } [ "," ] ] "}" ;
UseGlob      = PathHead "::" "*" ;
UseEntry     = Identifier [ "as" Identifier ] ;
```

Inline `mod` bodies reuse the same `{ HeaderItem } { Declaration }` shape, so
a nested module has its own header — nothing about the header rule changes
with depth. `Path` replaces `VariantPath` in `PrimaryExpression`, and
`NamedType` becomes a path so that `ui::hstack::HStack` parses in
annotations.

**Keyword ledger:** 42 − `from` − `import` + `mod` + `super` = **42**.
`mod` and `super` join the keyword table (and `super` leaves the reserved
list); `from` and `import` leave the keyword table and are *not* reserved —
nothing in the language claims them. The ambiguity note: every new position (`mod`, `super`,
`::` paths in the header) begins no token that was legal before, except the
hyphen rule, which is settled by the `::`-lookahead of
[Paths](#paths).

## Impact

The chapters named in [proposal.md](proposal.md), and what each gains:

| Chapter | Change |
|---------|--------|
| [`../../lexical-structure.md`](../../../lexical-structure.md) | keyword count 42 → 42 (`mod`, `super` in; `from`, `import` out), reserved list loses `super`, the `.`/`::` operator note; see [specs/lexical-structure.md](specs/lexical-structure.md) |
| [`../../grammar.md`](../../../grammar.md) | `HeaderItem`, `ModDecl`, `UseDecl`, `PubUse`, `Path`, `PathExpr`, `NamedType`, keyword list, index; see [specs/grammar.md](specs/grammar.md) |
| [`../../ast.md`](../../../ast.md) | header nodes replace the four import nodes, `PathNode`, `PathExpr`; see [specs/ast.md](specs/ast.md) |
| [`../../names.md`](../../../names.md) | module identity, namespace entity row, scope table, resolution order; see [specs/names.md](specs/names.md) |
| [`../../modules/README.md`](../../../modules/README.md) | packages as declared trees, index row; see [specs/modules-README.md](specs/modules-README.md) |
| [`../../modules/import-export.md`](../../../modules/import-export.md) | rewritten: `mod` declarations and layout, paths, the two verbs, type-only, collisions, manifest mapping; see [specs/import-export.md](specs/import-export.md) |
| [`../../modules/source-files.md`](../../../modules/source-files.md) | header/order rules, forms table, entry = tree root, file scope; see [specs/source-files.md](specs/source-files.md) |
| [`../../modules/visibility.md`](../../../modules/visibility.md) | package boundaries, initialization order and `E0611`, `E0604` rule, summary table; see [specs/visibility.md](specs/visibility.md) |
| [`../../expressions/path-and-access.md`](../../../expressions/path-and-access.md) | lanes: `::` paths and namespaces, `.` as value lane only, qualified paths without `use`; see [specs/path-and-access.md](specs/path-and-access.md) |
| [`../../type-system/checking-rules.md`](../../../type-system/checking-rules.md) | duplicates/visibility paragraph; see [specs/checking-rules.md](specs/checking-rules.md) |
| [`../../type-system/errors.md`](../../../type-system/errors.md) | `E0601`–`E0611` rows per the diagnostics table; see [specs/errors.md](specs/errors.md) |
| [`../../README.md`](../../../README.md) | §1.1 row for projects, workspaces, and the manifest format; see [specs/README.md](specs/README.md) |

Everything else — component syntax, generics, borrowing, concurrency,
patterns, the memory model — is untouched by construction: the change
rewrites how names *arrive* in a file and how files *become* modules, not
what any of them mean once there.
