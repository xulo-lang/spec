# README — Delta: Self-Hosting Note

> **Delta** — adds one informative paragraph to `README.md` § 1.1, between
> the out-of-scope table and the “Constructs that are absent …” paragraph;
> both neighbours are unchanged.

### 1.1 Out of scope

The following are deliberately **not** specified here:

| Excluded | Where it is documented |
|----------|------------------------|
| The standard library (`std/*`) and library-level APIs (JSON, regular expressions, I/O) | [xulo-website reference](https://xulo.org) |
| Tutorials, guides, and getting-started material | [xulo-website](https://xulo.org), `learn/book` in the implementation repository |
| The compiler/interpreter implementation, CLI, LSP, and rendering back ends | implementation repository (`xulo`) |
| Package registry protocol and tooling | xulo-website ecosystem documentation |
| Projects, workspaces, and the manifest file format itself — the language specifies packages and modules; how a project or workspace composes them, and the TOML file a manifest is, belong to tooling. The compiler consumes only the manifest's name mapping (dependency path name → module path), whose shape is specified in [`modules/import-export.md`](modules/import-export.md) | implementation repository (`xulo`), xulo-website ecosystem documentation |
| The machine-readable grammar files themselves (`xulo.ebnf`, `xulo.pest`, the tree-sitter grammar) — [`machine-readable.md`](machine-readable.md) links to them and is informative | implementation repository (`xulo`), xulo.org |

**Self-hosting note (informative).** A self-hosted implementation — a
compiler or tool written in Xulo itself — depends on services the language
does not provide: reading source files and command-line arguments, writing
to standard error, and choosing a nonzero exit code. The language supplies
none of them; a self-hosting tool reaches them through the standard
library's runtime-backed APIs or the FFI of [`ffi.md`](ffi.md) — for
example `extern "C" { fn exit(code: i32): unit }` on a C-ABI target. The
language itself pins only the exit status of ordinary termination: **0**
for a normal exit, nonzero for a runtime failure
([`memory-and-runtime.md`](memory-and-runtime.md)).

Constructs that are absent from the language rather than merely outside this
document's coverage — the language-level non-goals — are listed in
[`introduction.md`](introduction.md).
