# Xulo Language Specification — Delta: §1.1 Vocabulary Exclusion

> **Delta** — adds one row to the out-of-scope table of **1.1 Out of
> scope** in `README.md`.

| Excluded | Where it is documented |
|----------|------------------------|
| The standard library (`std/*`) and library-level APIs (JSON, regular expressions, I/O) | [xulo-website reference](https://xulo.org) |
| Tutorials, guides, and getting-started material | [xulo-website](https://xulo.org), `learn/book` in the implementation repository |
| The compiler/interpreter implementation, CLI, LSP, and rendering back ends | implementation repository (`xulo`) |
| Package registry protocol and tooling | xulo-website ecosystem documentation |
| Projects, workspaces, and the manifest file format itself — the language specifies packages and modules; how a project or workspace composes them, and the TOML file a manifest is, belong to tooling. The compiler consumes only the manifest's name mapping (dependency path name → module path), whose shape is specified in [`modules/import-export.md`](modules/import-export.md) | implementation repository (`xulo`), xulo-website ecosystem documentation |
| The machine-readable grammar files themselves (`xulo.ebnf`, `xulo.pest`, the tree-sitter grammar) — [`machine-readable.md`](machine-readable.md) links to them and is informative | implementation repository (`xulo`), xulo.org |
