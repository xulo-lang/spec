# Modules — Delta: Packages as Declared Trees

> **Delta** — replaces the introduction, **Module Identity**, and
> **Packages** sections of `modules/README.md`, and the `import-export.md`
> row of its **Index** table.

A module is a body of code — one file, or one inline `mod` block — together
with the names it exports: it is the unit of compilation, of visibility, and
of name resolution. Each module is checked on its own, sees its own
declarations plus the names it uses, and offers other modules exactly the
declarations it marks `pub`. This chapter defines what a source file
contains, how names cross from one module to another, and which names are
visible where.

## Module Identity

A module has no name of its own in the language: a module's identity is the
file or block that holds its body, and what names it is *reachable by* is
decided by the `mod` declarations that created it. Two files at different
paths are two modules even when their contents are identical, and the same
file claimed by one declaration is one module that is checked once.

Modules nest: a file's contents may open child modules (`mod fields;`
claiming `form/fields.xulo`, or an inline `mod tests { … }`), so a package
is a tree grown from its root file by declarations rather than a bag of
files discovered by path ([Packages](#packages)). The scope kinds that
exist *inside* a module are listed in [`../names.md`](../names.md).

## Exported and Private Names

Every declaration is module-private by default: a name declared in a module
can be used only inside that module. Marking a declaration `pub` exports it,
so that another module which uses it may refer to it:

```xulo
pub fn add(a: Int, b: Int): Int { a + b }   // exported
fn helper(n: Int): Int { n * 2 }            // private to this module
```

`pub` applies to module-level declarations (`fn`, `struct`, `enum`, `trait`,
`type`, `let`, `const`), to modules (`pub mod`), and to members (struct
fields and methods). Using a module binds only its exported names; a private
name is unreachable from outside its module by any route, including a glob,
a re-export, or a direct qualified path. The defaults, the member rules, and
the diagnostics that follow from breaking them are specified in
[visibility.md](visibility.md).

## Packages

A package is the tree of modules reachable from one root module by `mod`
declarations: start at the file the environment designates as the root,
follow every `mod` declaration in the tree — file-backed and inline alike —
and the modules those claim are the package. Membership is declared, not
discovered: a `.xulo` file that no declaration claims is not a module of the
package, is not checked, and is not nameable.

Paths cross package boundaries through the manifest mapping — a package head
names a dependency, and the environment supplies which file each head means
([import-export.md](import-export.md)). Project manifests themselves, the
package registry protocol, and package-management tooling are outside this
specification ([README §1.1](../README.md)).

## Index

| File | Contents |
|------|----------|
| [source-files.md](source-files.md) | File format, top-level declarations, the header and declaration order, the entry point, file scope versus block scope |
| [import-export.md](import-export.md) | `mod` declarations and layout, paths and manifest names, `use`/`pub use`, type-only use, collisions |
| [visibility.md](visibility.md) | Default privacy, `pub` on declarations and members, package boundaries, initialization, encapsulation, summary table |
