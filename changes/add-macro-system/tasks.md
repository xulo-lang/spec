# Tasks — add-macro-system

## Acceptance

- [ ] Fold the 14 modified-chapter deltas into the chapters their blockquotes
      name: `grammar.md`, `ast.md`, `introduction.md`,
      `type-system/errors.md`, `type-system/checking-rules.md`,
      `expressions/calls.md`, `testing.md`, `memory-and-runtime.md`,
      `types/traits.md`, `names.md`, `modules/import-export.md`,
      `modules/visibility.md`, `builtins/prelude.md`, and
      `builtins/README.md`.
- [ ] Add the six new chapters: `builtins/execution.md`, `builtins/context.md`,
      `builtins/sandbox.md`, `builtins/quota.md`, `builtins/errors.md`, and
      `builtins/cache.md`.
- [ ] After `grammar.md` changes, regenerate the external grammar snapshots
      (`xulo.ebnf`, `xulo.pest`, tree-sitter files) in the implementation
      repository; `machine-readable.md` is informative and lists those files
      rather than productions.
- [ ] Remove the draft row from `changes/README.md`, add an `accepted` archive
      row, and move the directory to `archive/`.

## Cross-references

- [ ] Check `builtins/README.md`, `builtins/prelude.md`, and the top-level
      `README.md` for stale “two layers” or “no macro system” wording.
- [ ] Check `types/traits.md` for remaining “closed registry” wording; the
      built-in registry is closed, the attribute namespace is not.
- [ ] Check `type-system/errors.md` for the `E1001`–`E1012` rows, the
      `E1001`–`E1099` range row, and any accidental reuse of `E0801`–`E0818`.
- [ ] Check `names.md`, `modules/import-export.md`, and
      `modules/visibility.md` for a consistent macro namespace: one key per
      fully qualified macro, `use` imports macro names, `pub` exports both the
      macro and the implementation function.
- [ ] Check every `specs/` file's first heading and `> **Delta** — …`
      blockquote against the `changes/README.md` convention.

## Implementation

- [ ] Parse the widened `AttributeArg` and the `MacroCall`/`QuoteCall`
      productions; extend `AttributeNode` with `AttributeArgNode`; add
      `MacroCallExpr` and `QuoteExpr` to the AST.
- [ ] Register `#[macro]` definitions before expansion; build the macro
      namespace; implement `use`/`pub use`/`pub` for macros; implement
      `E1010` definition checks.
- [ ] Implement the expansion loop: derive before attribute, same-kind source
      order, function macros in expression order, new calls in the next round,
      100-round limit (`E1004`), local-name resolution before expansion
      (`E1009`).
- [ ] Implement sandboxed compile-time execution: the value model, macro
      calls, context operations, `ast::quote!` AST construction, the
      `ast::call_*` entry points, quota accounting (`E1001`), sandbox checks
      (`E1007`), traps (`E1012`), and the span-chain diagnostics (`E1008`,
      `E1002`, `E1003`, `E1011`).
- [ ] Implement the structured-input cache with the key and quota replay of
      [specs/builtins/cache.md](specs/builtins/cache.md).
- [ ] Extend the prelude with the `ast` namespace and the macro records;
      reserve `ast` (`E0106`).
- [ ] Keep the built-in `Debug` derive and its `E0112` generic restriction;
      add user derive targets without changing `Display`/`ToString`
      reservation (`E0111`).
- [ ] Extend the attribute hosts to `type`, `let`, and `const`; keep `#[test]`
      on `fn` only.

## Tests

- [ ] Implementation tests: call a macro function directly from `#[test]` with
      constructed `ast::` input records; assert on the output records.
- [ ] Expansion tests: a fixture that invokes each kind and inspects the
      expanded AST, plus a compile-fail fixture for each diagnostic.
- [ ] Integration tests: a derived impl, an attribute macro that replaces a
      function, a function macro used in an expression, and a macro that
      calls another macro through an `ast::call_*` entry point.
- [ ] Error tests: `E1001`–`E1012`, including the span chain for a nested
      `ast::call_*`, a malformed `#[macro]`, a malformed quote, a sandbox
      violation, and a quota source note.
- [ ] Namespace tests: two modules defining the same short macro name;
      `use m1::w` vs `use m2::w`; `use m2::w as w2`; a macro and a function
      with the same spelling; a private macro from another module (`E0602`
      for `use`, `E1005`/`E0108` for invocation).
- [ ] Cache tests: two invocations with the same structured input (including
      spans), context observations, and macro implementation share an entry; a
      whitespace edit that shifts an observed span invalidates; a context
      answer change invalidates; a macro-implementation or compiler-version
      change invalidates; a cached expansion respects a lowered quota.

## Open follow-ups

- [ ] Keep the execution strategy, representation, and quota step mapping in
      the implementation documentation; the specification fixes only
      observable behaviour.
- [ ] Cache persistence and eviction policy.
- [ ] Macro debugging surface.
- [ ] Distribution of macro packages.
- [ ] Runtime behaviour of the `ast::call_*` entry points (currently a runtime
      panic).
- [ ] Per-fragment span annotations for `ast::quote!`.
