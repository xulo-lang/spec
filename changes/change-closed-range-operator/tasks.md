# Tasks: change-closed-range-operator

## Preflight

- [ ] No unaccepted dependency — every delta is written against the current,
      folded chapters.
- [ ] Confirm no chapter spells a closed range with any form other than `...`;
      `grep -rn '\.\.\.' --include='*.md' .` outside `changes/` separates
      spreads, block-comment ellipses, and the closed-range uses listed below.

## Fold

- [ ] Replace the operator token row and the longest-match section in
      [`specs/lexical-structure.md`](specs/lexical-structure.md).
- [ ] Replace the `SymbolOperator` row, `RangeExpression`, the spread/range
      paragraph, and `RangePattern` in
      [`specs/grammar.md`](specs/grammar.md).
- [ ] Replace the `RangeExpr.op` union and the two prose sentences in
      [`specs/ast.md`](specs/ast.md).
- [ ] Replace the `Range<T>` section in
      [`specs/types/primitive-types.md`](specs/types/primitive-types.md).
- [ ] Replace the range-pattern example in
      [`specs/types/enums.md`](specs/types/enums.md).
- [ ] Replace the precedence row, the spread note, and the Ranges section in
      [`specs/expressions/operators.md`](specs/expressions/operators.md).
- [ ] Replace the precedence row and the spread note in
      [`specs/expressions/README.md`](specs/expressions/README.md).
- [ ] Replace the range-pattern rows, the two examples, and the Ranges section
      in [`specs/expressions/control-flow.md`](specs/expressions/control-flow.md).
- [ ] Replace the range operator row and the range-pattern row in
      [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md).
- [ ] Replace the `Range<T>` row in
      [`specs/builtins/prelude.md`](specs/builtins/prelude.md).
- [ ] Replace the output-table row, the `asin`/`acos` bound, and the
      `to_fixed` digits bound in
      [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md).
- [ ] Replace the closed-range example and its prose in
      [`specs/introduction.md`](specs/introduction.md).
- [ ] Replace the `for` example in
      [`specs/components/state.md`](specs/components/state.md).
- [ ] Replace the `to_fixed` failure-table row in
      [`specs/error-handling.md`](specs/error-handling.md) and
      [`specs/memory-and-runtime.md`](specs/memory-and-runtime.md).

## Neighbouring edits

- [ ] `expressions/literals.md` — ranges are not literals and the spread stays
      `...`; no edit.
- [ ] `types/composite-types.md`, `expressions/literals.md` — spread examples
      keep `...`; no edit.
- [ ] `ast.md` `RangePattern.endInclusive` stays a `boolean`; only the prose
      naming the operators changes.
- [ ] `changes/archive/**` — historical records are not rewritten.

## Sweep

- [ ] `grep -rn '0\.\.\.=15' --include='*.md' .` outside `changes/` — empty.
- [ ] `grep -rn 'a\.\.\.b\|0\.\.\.3\|1\.\.\.n\|90\.\.\.100\|1\.\.\.5\|0\.\.\.29\|30\.\.\.69\|-1\.0\.\.\.1\.0\|start\.\.\.end\|lo\.\.\.hi' --include='*.md' .`
      outside `changes/` — empty.
- [ ] `grep -rn 'closed-range operator\|closed range' --include='*.md' .`
      outside `changes/` — every hit names `..=`, never `...`.
- [ ] `grep -rn '`\.\.<` `\.\.\.`\|`\.\.<` `\.\.=`' --include='*.md' .`
      outside `changes/` — every precedence and operator table lists `..<`
      and `..=`.
