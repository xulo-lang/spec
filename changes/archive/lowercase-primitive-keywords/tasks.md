# Tasks: lowercase-primitive-keywords

## Preflight

- [x] No unaccepted dependency — the deltas are written against the current,
      folded chapters. (`align-intrinsic-casing` is independent: it renames
      free prelude functions, not type names.)

## Fold — structural deltas

- [ ] Keyword count + Primitive types category in
      [`specs/lexical-structure.md`](specs/lexical-structure.md) (58 in both
      ledgers; `null` row moves out of Literals).
- [ ] Grammar ledger: the count sentence in `grammar.md` becomes 58, and the
      `Keyword` production gains the seventeen spellings (its prose currently
      says 42 against a 41-entry list — the count is corrected here).
- [ ] Two-groups bullet + shadowing paragraph in
      [`specs/names.md`](specs/names.md) (`Result` and `Error` join the
      SHOULD-NOT-shadow list).
- [ ] Reserved-names table in
      [`specs/builtins/prelude.md`](specs/builtins/prelude.md) (base-type
      row removed; `Null`/`Unit`/`Unknown` removed from the value row).
- [ ] Intro of [`specs/types/primitive-types.md`](specs/types/primitive-types.md).

## Fold — global rename

- [ ] Apply the migration table to **every chapter** (all `*.md` at the repo
      root and in subdirectories, excluding `changes/`): rename the old
      spellings `Boolean` `String` `Number` `Int` `Float` `I8` `I16` `I32`
      `I64` `U8` `U16` `U32` `U64` `F32` `F64` `Unit` `Unknown` to `bool`
      `string` `number` `int` `float` `i8` `i16` `i32` `i64` `u8` `u16`
      `u32` `u64` `f32` `f64` `unit` `unknown` **only inside** fenced code
      blocks, inline code spans, and table cells — never in English prose.
- [ ] The script MUST use word boundaries and MUST NOT touch AST node names
      (`StringLiteral`, `NumberLiteral`, `BooleanLiteral`, `LiteralNode`)
      or the protocol `ToString` / method `toString` — `\bString\b` etc.
      never match inside those, and the check below proves it.
- [ ] `View` and `Range` are NOT in the table and MUST remain capitalized
      everywhere.

## Sweep

- [ ] `grep -rnE '\b(Boolean|Number|Int|Float|I8|I16|I32|I64|U8|U16|U32|U64|F32|F64|Unit|Unknown)\b' --include='*.md' . | grep -v changes/` returns nothing. (`String` needs a
      human read: remaining matches must all be `ToString`, `StringLiteral`,
      or English prose such as "string" inside a larger word — verify each.
- [ ] `grep -rnE '\bbool\b' types/primitive-types.md` — the section heading
      and every annotation use the keyword.
- [ ] Both ledgers: `grep -c` over the `Keyword` production and the
      lexical table sums to 58; the two prose counts agree.
- [ ] `grep -rn 'ToString\|toString' --include='*.md' . | grep -v changes/`
      — every hit is the protocol or method, never a mangled `Tostring`.
- [ ] English spot-check: `grep -rn 'the number of' builtins/prelude.md`
      still reads as English (prose untouched).

## Consistency

- [ ] `names.md`: primitives listed as keywords; `List` `Map` `Set` `Task`
      `Range` `View` `Result` `Error` in the SHOULD-NOT-shadow bullet.
- [ ] `builtins/prelude.md`: reserved table has four rows; the `ToString`
      bullet and `str` section use `bool` `string` `int` `float`.
- [ ] `type-system/errors.md`: no message quotes an old primitive spelling
      (`E0212`'s trigger text is rename-through if it names types).
- [ ] `ast.md` untouched except prose that spells source types; node names
      intact.
- [ ] Link check across the folded files.

## Acceptance

- [ ] All boxes above checked; archive the directory as `accepted`; move the
      index entry in [`../changes/README.md`](../../README.md).
