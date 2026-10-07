# Modules

A module is one source file together with the names it exports: it is the unit
of compilation, of visibility, and of name resolution. Each `.xulo` file is
checked on its own, sees its own declarations plus the names it imports, and
offers other files exactly the declarations it marks `pub`. This chapter
defines what a source file contains, how names cross from one file to another,
and which names are visible where.

## Module Identity

A module has no name of its own: a module's identity is its file path. Two
files at different paths are two modules even when their contents are
identical, and the same file reached by two different specifiers is one module
that is checked once. Consequently a file declares no module header and no
module keyword — the first token of a file is already a declaration.

There are no block-level modules. A `{ … }` block, a function body, a
`struct`, an `impl`, and a component body never introduce a module: one file
is one module, always. The scope kinds that *do* exist inside a file are
listed in [`../names.md`](../names.md).

## Exported and Private Names

Every declaration is module-private by default: a name declared in a file can
be used only inside that file. Marking a declaration `pub` exports it, so that
another module which imports it may refer to it:

```xulo
pub fn add(a: int, b: int): int { a + b }   // exported
fn helper(n: int): int { n * 2 }            // private to this file
```

`pub` applies to module-level declarations (`fn`, `struct`, `enum`, `trait`,
`type`, `let`, `const`) and to members (struct fields and methods). Importing
a module brings in only its exported names; a private name is unreachable
from outside its file by any route, including a namespace import or a
re-export. The defaults, the member rules, and the diagnostics that follow
from breaking them are specified in [visibility.md](visibility.md).

## Packages

A package is the set of modules reachable from one entry file: start at the
entry file, follow every import specifier it uses, take the modules those
resolve to, and repeat transitively. Package boundaries are therefore decided
by path and visibility alone — a module belongs to the package when some chain
of imports reaches its file, and what each link of that chain may see is
decided by `pub`.

How a specifier becomes a file is specified in
[import-export.md](import-export.md). The package registry protocol and
package-management tooling are outside this specification
([README §1.1](../README.md)).

## Index

| File | Contents |
|------|----------|
| [source-files.md](source-files.md) | File format, top-level declarations, declaration order and forward references, the entry point, file scope versus block scope |
| [import-export.md](import-export.md) | The four import forms, exports and re-exports, specifier resolution, cycles, collisions |
| [visibility.md](visibility.md) | Default privacy, `pub` on declarations and members, package boundaries, encapsulation, summary table |
