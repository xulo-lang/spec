# Tasks — Add Foreign Function Interface

**Status:** accepted · **Date:** 2026-10-21
**Change ID:** add-foreign-function-interface

How to complete this change: work top to bottom; every item that edits a
chapter edits it through the delta in `specs/` of the same name, so review
can compare delta against chapter. Nothing here is committed to the main
chapters until the change status moves past draft.

## Fold the deltas (at acceptance)

- [x] `specs/ffi.md` → new chapter `ffi.md`; the README document map row is
  already part of `specs/README.md` (fold that delta together with it so the
  map never points at a missing file).
- [x] `specs/lexical-structure.md` → `lexical-structure.md` (keyword count
  sentence, three table rows).
- [x] `specs/grammar.md` → `grammar.md` (count sentence, `Keyword`,
  `Declaration`, ambiguity note, six new productions, Declarations prose,
  production-index row).
- [x] `specs/ast.md` → `ast.md` (`DeclarationNode` union, three node
  interfaces after `FnDeclaration`).
- [x] `specs/introduction.md` → `introduction.md` (replace the non-goal
  bullet).
- [x] `specs/errors.md` → `type-system/errors.md` (ranges-table row; new
  `### Foreign boundary errors` subsection before `## Warnings`).
- [x] `specs/primitive-types.md` → `types/primitive-types.md`
  (**Pointer-width integers** subsection, two C-mapping rows, reserved-list
  `Ptr`, closing-paragraph clause).
- [x] `specs/memory-and-runtime.md` → `memory-and-runtime.md` (carve-out
  sentence in the zero-UB claim; "three" → "four" plus the new
  implementation-defined bullet).
- [x] `specs/README.md` → `README.md` (§ 1 bullet, § 4 map row, § 5 three
  terminology entries).
- [x] `specs/functions.md` → `functions.md` (one Definition bullet).

## Cross-reference checks (after folding)

- [x] Keyword count reads **61** in `lexical-structure.md` and `grammar.md`,
  and nowhere in the main chapters still says 58 (the archived change
  `archive/lowercase-primitive-keywords/` keeps its historical 41 → 58 and
  MUST NOT be edited).
- [x] Every `E090x` code mentioned in `ffi.md` exists in the errors.md
  ranges table and subsection, and no `E09xx` code outside `E0901`–`E0903`
  appears anywhere.
- [x] The production index of `grammar.md` lists all six `Extern*`/`AbiTag`
  nonterminals, and `Keyword` matches the lexical table row for row.
- [x] `README.md` document map row and the three terminology entries link to
  the new `ffi.md` and resolve.
- [x] `introduction.md` non-goal bullet is the narrower replacement; the
  neighbouring "Not a replacement for a systems language" bullet still
  matches the `Ptr`-never-dereferenceable line in `ffi.md`.
- [x] `memory-and-runtime.md` implementation-defined list counts four, and
  the "two categories" sentence before it was not disturbed.
- [x] `types/primitive-types.md` reserved list reads `…, `View`, `Error`,
  `Ptr`` and its closing paragraph names `ffi.md`.
- [x] `ast.md` node interfaces have no field the grammar does not admit
  (no `async`, `genericParams`, `where`, `attributes`, `isPub` on
  `ExternFunctionDeclaration`) and `ParameterNode.mode`/`default` are
  `null` for extern params.
- [x] Run the example blocks of `ffi.md` against the checker's test
  conventions: accepted forms annotated `// ok`, rejected forms either
  `// error[E090x]` for checker codes or `// error: reason` for syntax
  failures with no code.

## Verification

- [x] `grep -rn "extern" --include="*.md"` outside `changes/` returns hits
  only in the folded chapters (`ffi.md`, `grammar.md`, `lexical-structure.md`,
  `ast.md`, `introduction.md`, `functions.md`, `README.md`,
  `memory-and-runtime.md`, `type-system/errors.md`,
  `types/primitive-types.md`).
- [x] No main-chapter text contradicts the two rules that hold at every stage
  (`changes/README.md`): nothing in the unaccepted chapters assumes `extern`
  exists, and the deltas do not contradict the chapters they name.
- [x] On acceptance: fold in the order above, drop the index row of
  `changes/README.md`, move this directory to
  `changes/archive/add-foreign-function-interface/` with status `accepted`.
