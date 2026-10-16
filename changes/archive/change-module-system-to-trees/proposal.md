# Change Module System to Trees

**Status:** accepted · **Date:** 2026-10-16 · **Affects:** `lexical-structure.md`,
`grammar.md`, `ast.md`, `names.md`, `README.md`, `modules/README.md`,
`modules/import-export.md`, `modules/source-files.md`, `modules/visibility.md`,
`expressions/path-and-access.md`, `type-system/checking-rules.md`,
`type-system/errors.md`

## Summary

The module system is re-founded on a four-level vocabulary and a
declaration-built module tree:

```text
project      — the buildable unit: xulo.toml with dependencies and aliases
workspace    — several projects composed (tooling level)
package      — a tree of modules reachable from one root by `mod` declarations
module       — a file, or a block inside a file: one file may hold several
```

Modules are declared, not assumed — `mod form;` claims `form.xulo`, an inline
`mod tests { … }` nests inside a file — and names cross boundaries along
paths:

```xulo
// main.xulo — the tree root of an executable package
mod form;
mod internal;

use lsp-server::router::Router;
use ui::{HStack, Button};
use form::validate;
use self::internal::cache;
use type ui::User;

pub use ui::hstack::HStack;      // bind and re-export in one line (facade)
```

`use` is the verb that brings names into scope — singly, grouped, or by
glob (`use lsp-server::*`) — and `pub use` extends it with the interface, so
the bind-and-export cell that today costs two declarations becomes one line.
There is no third verb: initialization *between* packages is ordered by the
environment (dependencies before dependents) and is never declared in
source. Package names come from the project manifest: each dependency
resolves under exactly one path name — its `alias` when set, otherwise its
key — so no string literal appears anywhere in the module system. External
packages are addressed directly — `lsp-server::…`, `ui::…` — with
no `crate` prefix and no per-file binding line.

Until this change is accepted, the chapters remain normative: a file is a
module, specifiers are string literals, `import { … } from "…"` and
`pub use { … } [from "…"]` are the crossing forms
([`../../modules/import-export.md`](../../../modules/import-export.md)), and the
deltas in [`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The present syntax is assembled from three languages and belongs to none

Xulo today borrows the verb from ECMAScript (`import`), the export keyword
from Rust (`pub use`), the semantics of the forward form from ECMAScript
(`export {x} from "m"` binds nothing), and the position rule from neither
(imports are header-bound, `pub use` may appear anywhere). The result is a
table in which every cell comes from a different column:

| Form today | Verb from | Form from | Semantics from | Position from |
|---|---|---|---|---|
| `import { x } from "m"` | ES | ES | ES | Go/Java/Dart (header) |
| `pub use { x }` | Rust | Rust-ish | ES `export {x}` | Rust (anywhere) |
| `pub use { x } from "m"` | Rust | Rust + ES `from` | ES `export … from` (bindless) | Rust (anywhere) |

The word `use` never does what Rust's `use` means: neither `pub use` form
binds a name — the keyword was borrowed without its meaning. A comparison of
twelve languages (`design.md` carries the table) says the systems-language
camp — Rust, Swift, C++20, Scala — spells imports `use`, always binds when it
re-exports, and has no bindless forward form; the ECMAScript camp has the
opposite on every count. Xulo's own model — explicit interfaces, path-gated
visibility, a manifest-fed package environment — is a systems model wearing
web syntax.

### The four cells have no one-line answer

| | binds locally | binds nothing |
|---|---|---|
| **private use** | `import { x } from "m"` | (`import "m"` — initialization only) |
| **bind and export** | *two declarations today:* `import` + `pub use { x }` | — |
| **export only** | — | `pub use { x } from "m"` (forward), `pub use { x }` (own) |

The most common façade line — bring a name in *and* put it in the interface
— has no single form; the bindless forward, which exists only in the
ECMAScript lineage, is the one cell that does. The multi-language comparison
in [`design.md`](design.md#the-four-cells-in-other-languages) shows the
engineering languages solved this the other way: `pub use path` is one line,
and forwarding comes with a binding.

### "Package" has two incompatible definitions in one specification

`modules/README.md` defines a package as *the set of modules reachable from
one entry file* — a reachability closure that swallows every third-party
module the entry happens to use — while `import-export.md` speaks of "the
program's packages" in the plural, resolved by the environment. The two
cannot both hold: if reachable-is-packaged, there is one bag; if the
environment names several packages, membership was never about reachability.
A package that is *distributed* — the meaning every ecosystem uses — must be
a declared thing, not an emergent one.

### The concepts the model needs are missing or reserved

There is no `mod` (the word `module` sits in the reserved list, `mod` is an
ordinary identifier), `crate` does not exist and should not, a file is
forcibly one module with no way to nest, and the four-level hierarchy —
project, workspace, package, module — appears nowhere even though every
sibling ecosystem (Cargo, go.mod, npm) organizes itself by it.

## Scope

### In scope

- **Vocabulary.** Project, workspace, package, and module are named; the
  package is redefined as the tree of modules reachable from a root by `mod`
  declarations; projects and workspaces are excluded from the specification
  in §1.1 and recorded as tooling vocabulary.
- **Declaration-built trees.** `mod name;` claims a file (Rust-style nested
  layout: a sibling `form.xulo`, then `form/…` for its children), inline
  `mod name { … }` nests a module in a file, `pub mod` puts a namespace in
  the interface; a file is no longer automatically a module, and a file may
  be the body of at most one module declaration (`E0610`).
- **Paths.** A single path syntax (`::`) for package heads, module chains,
  `self`/`super`, variants, and associated names; paths resolve from the
  package root by default; qualified calls may appear without a preceding
  `use`.
- **Manifest name mapping.** The dependency table of `xulo.toml` provides the
  compiler a mapping from real package names to path names: exactly one per
  dependency — its `alias` when set (which replaces the key in source), its
  key when the key is writable in a path; the specification defines the
  shape of that mapping, not the manifest format.
- **Verb system.** `use` with plain, grouped, glob, type-only, and aliased
  forms, header-bound; `pub use` as bind-and-export; package initialization
  stays with the environment, so no side-effect form exists; `import` and
  `from` leave the keyword table along with the string specifier.
- **Operator lanes.** `::` owns the declaration lane (paths, namespace
  members including the built-in namespaces, variants, associated names);
  `.` owns the value lane (fields, methods, map entries, positions).
- **Diagnostics.** `E0603` is retired (its reason — no topological
  initialization order — no longer exists), `E0605`/`E0606`/`E0607` are
  re-scoped, `E0610` and `E0611` are added (`E0611` guards an initializer
  that reads or calls into a module that has not initialized yet),
  `E0104` gains the manifest and glob cases, and `E0604` is reworded so the
  façade case is decided by the text.

### Out of scope

- **`crate` or any absolute-path keyword** — bare paths are already
  package-root-absolute, and package heads are disambiguated by the manifest
  mapping ([design.md](design.md#no-crate)).
- **Package-level visibility (`pub(crate)` and friends)** — the language
  keeps exactly two levels, `pub` and private; visibility crosses module
  boundaries, not package tiers.
- **Projects and workspaces as language constructs** — their manifests and
  composition stay tooling ([design.md](design.md#the-four-level-vocabulary)).
- **Directory/index modules, `mod.rs`, or a magic entry file** — layout is
  the lookup rule of `mod` and nothing else.
- **The manifest format itself** — TOML shape, versions, sources, and
  registry protocol remain tooling; only the mapping the compiler consumes is
  specified.
- **Macros, `extern crate`-style declarations, and per-file package
  aliasing** — dependency names are global to the package through the
  manifest; a file may locally rename a path with `as`, but may not declare
  dependencies.
- **Relaxing type-only erasure or the component family** — `use type`
  preserves today's erasure semantics; `@`-state declarations are untouched.

## Compatibility

- **This is a breaking replacement, folded as a whole.** Every module-system
  form changes spelling: `import { x } from "m"` becomes `use m::x` (or a
  manifest head plus a path), string specifiers disappear, and files stop
  being modules until a `mod` declaration claims them. The specification is
  a pre-1.0 draft whose whole is not frozen
  ([`../../README.md`](../../../README.md), §2), and the workflow requires a
  change to be folded in whole or not at all
  ([`../README.md`](../../README.md)).
- **All examples migrate in the same fold.** The sweep is enumerated in
  [tasks.md](tasks.md); no chapter is left speaking the retired syntax.
- **Diagnostics move with the syntax.** A program can only be affected by
  writing the new forms — every new path, declaration, or manifest head
  rules through the codes listed above.

The swap, side by side — one local module under both systems, and what a
dependency looks like once strings are gone:

```xulo
// today — a file is a module; names arrive through strings
import { validate } from "form"
import "core"                 // a third form whose only job was ordering

let ok = validate(input)
```

```xulo
// after — the tree declares its files; names arrive through paths
mod form;                      // claims form.xulo next to this file
use form::validate;

let ok = validate(input)
```

```toml
# after — each dependency's path name lives in the manifest, once
[dependencies]
lsp-server = "0.8"
clap       = { version = "4.6.6", alias = "clap_lib" }
```

```xulo
use lsp-server::router::Router;
use clap_lib::Command;         // the key `clap` is manifest-internal now
```

The pair shows what prose cannot: the file stops being a module until
`mod` claims it, and `import "core"` has no replacement line — inside the
tree the order follows the `mod` declarations (post-order, siblings in
declaration order), between packages it is the environment's contract
([design.md](design.md#initialization-and-the-retirement-of-e0603)).

## Non-goals

- **Keeping string specifiers "for compatibility"** — they are the very
  split (two lexical shapes for one job) the change removes
  ([design.md](design.md#string-specifiers-in-source--rejected)).
- **A `use … from` hybrid** — ES grammar with a Rust verb was tried in the
  discussion and rejected as another dialect rather than a fix
  ([design.md](design.md#use-pub-use--recommended)).
- **Bindless re-export kept as a separate form** — the engineering languages
  live without it; the façade needs `pub use`, not a non-binding edge
  ([design.md](design.md#the-four-cells-in-other-languages)).
- **Implicit local modules (file-as-module) kept alongside declarations** —
  two ways to become a module is the same class of split the change is
  closing ([design.md](design.md#building-the-tree)).

## Readiness

- [x] Alternatives and chosen design — twelve-language comparison, four-cell
      analysis, `crate` replacement, manifest mapping, verb system, operator
      lanes, initialization order — [design.md](design.md)
- [x] Core module chapters —
      [specs/import-export.md](specs/import-export.md),
      [specs/source-files.md](specs/source-files.md),
      [specs/visibility.md](specs/visibility.md),
      [specs/modules-README.md](specs/modules-README.md)
- [x] Language surface —
      [specs/lexical-structure.md](specs/lexical-structure.md),
      [specs/grammar.md](specs/grammar.md),
      [specs/ast.md](specs/ast.md),
      [specs/names.md](specs/names.md),
      [specs/path-and-access.md](specs/path-and-access.md)
- [x] Checking and diagnostics —
      [specs/checking-rules.md](specs/checking-rules.md),
      [specs/errors.md](specs/errors.md)
- [x] §1.1 vocabulary exclusion — [specs/README.md](specs/README.md)
- [x] The example sweep and neighbouring passages — [tasks.md](tasks.md)
- [x] Initialization soundness — settled: `E0611` for direct reads and
      direct calls from file-scope initializers —
      [design.md](design.md#initialization-and-the-retirement-of-e0603)
- [x] `E0610`/`E0611` and the re-scoped `E0605`–`E0607` registered in
      [`../../type-system/errors.md`](../../../type-system/errors.md) at
      acceptance, exactly as the codes named by
      [specs/errors.md](specs/errors.md) become chapter text.
