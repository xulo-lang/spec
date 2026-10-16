# Type Errors and Diagnostics — Delta: Module-Family Codes

> **Delta** — replaces the `E0104` row of **Name and declaration errors**,
> the whole **Module and visibility errors** table, the `W0102` row of
> **Warnings**, and the `W0102` gallery example in `type-system/errors.md`.

The name and declaration row:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0104` | a collision between bindings header forms create: two `use` entries, a `use` or `mod` name against a declaration, a glob overlap, a manifest path name against a local `mod` or alias | `use math::add;` then `use util::add` | `` `add` is bound twice `` | [import-export.md](../modules/import-export.md) |

The module and visibility table:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0601` | a path names a member the target module does not export | `use math::nope` | ``module `math` does not export `nope` `` | [import-export.md](../modules/import-export.md) |
| `E0602` | a name exists but is not visible here | `use math::helper` with private `helper` | `` `helper` is private to `math` `` | [import-export.md](../modules/import-export.md) |
| `E0604` | a `pub` signature mentions a type that is not part of the declaring module's interface — not `pub` there, not exported there by a `pub use` chain, not universally visible | `pub fn load(): Cache` | ``private type `Cache` in public interface`` | [visibility.md](../modules/visibility.md) |
| `E0605` | a header form — `mod`, `use`, `pub use` — is written outside the file's header | a `use` after a top-level `fn` | ``module and use declarations must appear at the top of the file`` | [source-files.md](../modules/source-files.md) |
| `E0606` | a type-only binding — declared by `use type`, or propagated through a `pub use` chain — is used where a value is required | `let u = User` after `use type ui::User` | `` `User` is a type-only binding and is not a value `` | [import-export.md](../modules/import-export.md) |
| `E0607` | a path head resolves to nothing: an unknown package name, a missing `mod` file, `super` above the package root | `mod shapes;` with no `shapes.xulo` | ``cannot resolve `shapes` `` | [import-export.md](../modules/import-export.md) |
| `E0608` | the package's root module declares no `main` | a root module without `main` | ``entry module does not declare `main` `` | [source-files.md](../modules/source-files.md) |
| `E0609` | `main` has parameters or any return type but `Unit` or `View` | `fn main(argc: Int)` | `` `main` must take no parameters `` | [source-files.md](../modules/source-files.md) |
| `E0610` | a `mod` declaration resolves to a file that is already the body of another module, or to the file that declares it | root `form.xulo` declaring `mod form;` | ``file `form.xulo` is the body of more than one module`` | [import-export.md](../modules/import-export.md) |
| `E0611` | a file-scope initializer directly reads a file-scope `let` of a module that initializes after it, or directly calls a function declared in such a module | `let snap = config::N` in `stats.xulo`, with `mod config` ordered after `mod stats` | `` `config::N` is used before `config` initializes `` | [visibility.md](../modules/visibility.md) |

`E0603` — the import-cycle error — is **retired and reserved**: with the
declaration-built tree, initialization order is structural, so no cycle can
prevent an order from existing, and the number is never reused
([visibility.md](../modules/visibility.md)).

The warning row:

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `W0102` | a `use` entry is never used, or a glob contributes no name that is used | `use math::add` unused | ``unused `use` of `add` `` | [import-export.md](../modules/import-export.md) |

And its gallery example:

```text
warning[W0102]: unused `use` of `add`
 --> app.xulo:1:10
  │
1 │ use math::add;
  │          ^^^ this name is never used
```
