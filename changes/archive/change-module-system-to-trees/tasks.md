# Tasks — fold checklist for `change-module-system-to-trees`

Counts below were measured on the pre-fold tree and shrink as the sweep
proceeds; the post-fold greps are the acceptance signal.

## Preflight (before folding)

- [x] Fold order with `add-declaration-attributes`: that draft's
      `specs/lexical-structure.md` also replaces the operators table (adds
      the `#` row). Fold one change, then re-apply the other's table rows —
      do not take both full tables verbatim.

## Fold (whole or not at all)

- [x] Fold each delta into its chapter: `specs/import-export.md` →
      `modules/import-export.md`; `specs/source-files.md` →
      `modules/source-files.md`; `specs/visibility.md` →
      `modules/visibility.md`; `specs/modules-README.md` →
      `modules/README.md`; `specs/path-and-access.md` →
      `expressions/path-and-access.md`; `specs/checking-rules.md` →
      `type-system/checking-rules.md`; `specs/errors.md` →
      `type-system/errors.md`; `specs/lexical-structure.md` →
      `lexical-structure.md`; `specs/grammar.md` → `grammar.md`;
      `specs/ast.md` → `ast.md`; `specs/names.md` → `names.md`;
      `specs/README.md` → `README.md` (§1.1 row).
- [x] Post-fold leftover check: in each folded chapter, grep for the
      retired spellings — `from "`, `import {`, `import * as`,
      `import type`, `pub use .* from`, `mod.rs`, `index.xulo` — and read
      any section the delta did not replace for stale examples.

## Example sweep (after folding)

- [x] **String specifiers — 40 sites, 10 files.** 39 `from "…"` clauses
      plus 1 side-effect `import "…"`: `ast.md`,
      `components/environment.md`, `expressions/path-and-access.md`,
      `introduction.md`, `modules/import-export.md`,
      `modules/source-files.md`, `modules/visibility.md`, `names.md`,
      `type-system/errors.md`, `types/traits.md` (the module chapters are
      covered by their deltas — re-read for residual examples). Rewrite to
      manifest heads with paths: `use form::validate`, `use lsp-server as
      ls`; the side-effect line `import "…"` retires outright — a package
      initializes from the manifest and the environment, so the statement's
      job disappears with it.
- [x] **Bindless forms — 37 occurrences.** `import type { T }` → `use type`,
      `import { a, b }` → grouped `use`, `import * as ns` → `use m as ns`
      or a glob.
- [x] **Namespace dots — 153 sites, 14 files.** `Math.` `Task.` `Time.`
      `env.` (…) → `::`: `builtins/intrinsic-functions.md` (109),
      `expressions/async-expressions.md` (11), `error-handling.md` (8),
      `expressions/path-and-access.md` (5), `memory-and-runtime.md` (4),
      `concurrency.md` (4), `testing.md` (3), `types/function-types.md`
      (2), `type-system/coercion.md` (2), then one each in `names.md`,
      `lexical-structure.md`, `functions.md`, `builtins/prelude.md`,
      `builtins/README.md`.
- [x] **Trait-dispatch dots — 18 sites, 5 files.** `Area.area(r)` →
      `Area::area(r)` (and `Comparable.` `Clone.` `Eq.` `Ord.`
      `ToString.`): `types/traits.md` (9), `types/generics.md` (3),
      `types/function-types.md` (3), `functions.md` (2),
      `expressions/path-and-access.md` (1).
- [x] **Dotted module members** — `expressions/calls.md:16` (`math.add`)
      and any residual `form.` / `math.` calls; the `E0602` rows in
      `type-system/errors.md` and `modules/import-export.md` are already
      reworded by the deltas.
- [x] **Post-fold greps return zero** outside the retirement notes:
      `from "`, `import {`, `import * as`, `import type`, `import "`,
      any `import` statement in a code sample (`^\s*import\b`),
      `\bMath\.`, `\bTask\.`, `\bTime\.`, `\bArea\.`, `import cycle`,
      `import graph`. `E0603` appears only in the retirement sentences
      (`type-system/errors.md`, `modules/visibility.md`).

## Neighbouring passages (manual — not in any delta)

- [x] `introduction.md:233` — module-system comparison row: replace
      "A file is a module; … `import { a } from "m"`" with the
      declaration-built tree and `use m::x`; also `:236` (the `import`
      item in the shared-surface list) and re-read `:240` for consistency.
- [x] `builtins/README.md:44` — scope-order item 4 says "names brought into
      the module by `import` and by `pub use`": align with the
      `names.md` delta ("`mod`, `use`, and `pub use`").
- [x] `README.md:168` — intrinsic glossary: "available without import" →
      "available without `use`".
- [x] `type-system/errors.md:267` — E0101 gallery note "or import it" →
      "or `use` it".
- [x] `type-system/errors.md:110` — E0102 row still says "namespace
      member": keep (variant `Color::Redd` is unchanged) or reword to
      "field, method, or variant".
- [x] `machine-readable.md` — the linked grammar files live in the
      implementation repository (§1.1): leave a note there that
      `xulo.ebnf` / the tree-sitter grammar need regeneration from the new
      `grammar.md`.

## Consistency checks

- [x] Keyword ledger is 42 in `lexical-structure.md`, `grammar.md`, and
      design.md; reserved list contains `mod` and not `super`; `from` and
      `import` appear in no keyword table and are not reserved.
- [x] One path name per dependency reads consistently: `alias` replaces the
      key in `design.md` + `specs/import-export.md`, the `E0607` row covers
      a path spelling a replaced key, and no text anywhere implies a
      dependency is reachable under two names.
- [x] Diagnostics agree three ways: design.md table ↔ `specs/errors.md`
      rows ↔ folded `type-system/errors.md` — `E0601` `E0602` `E0604`
      `E0605` `E0606` `E0607` `E0608` `E0609` `E0610` `E0611`, `E0603`
      retired-and-reserved, `E0104` widened, `W0102` reworded — one message
      shape per code.
- [x] Grammar/AST agreement: `HeaderItemNode` union in `ast.md` matches
      `HeaderItem` in `grammar.md`; `PathExpr`/`PathNode` used consistently;
      the index tables of `grammar.md` list every new nonterminal.
- [x] Cross-links: every `changes/` link and every as-if-folded link in the
      deltas resolves (the pre-fold check passed — re-run after any edit).

## Acceptance

- [x] Proposal readiness boxes all checked (after preflight and review),
      `changes/README.md` index updated, delta headers still name their
      targets.
