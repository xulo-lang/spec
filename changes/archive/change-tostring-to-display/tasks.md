# Tasks: change-tostring-to-display

## Preflight

- [x] No unaccepted dependency — the deltas are written against the current,
      folded chapters; the registry text they replace comes from
      `archive/add-declaration-attributes/`, already folded.
- [x] No grammar, lexical, or AST file is named by any delta — identifiers
      only, production count untouched.

## Fold

- [x] [`specs/builtins/prelude.md`](specs/builtins/prelude.md) — opening
      sentence, `Format` types row, the whole "Built-in protocols" section,
      the two reserved-name rows plus the `ToString` reservation note.
- [x] [`specs/builtins/README.md`](specs/builtins/README.md) — the Prelude
      layer row and the `prelude.md` index row.
- [x] [`specs/builtins/intrinsic-functions.md`](specs/builtins/intrinsic-functions.md) —
      the first `print` bullet, the consult paragraph, the precedence
      table, the concrete-timing paragraph, the output table's completion
      rows, the struct example, the whole "Conversion" section, the `str`
      index row.
- [x] [`specs/types/traits.md`](specs/types/traits.md) — the final
      "Structural vs nominal" bullet, the new "Built-in formatting traits"
      section (paste before "## Attributes"), the two `#Derive` examples and
      the registry bullet, the whole "### Derive" section.
- [x] [`specs/types/composite-types.md`](specs/types/composite-types.md) —
      the `join` bullet.
- [x] [`specs/types/enums.md`](specs/types/enums.md) — the interpolation
      bullet.
- [x] [`specs/type-system/coercion.md`](specs/type-system/coercion.md) — the
      last coercion-table row and the whole "string interpolation" section.
- [x] [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md) —
      the template-literal row and the `${e}` well-formedness bullet.
- [x] [`specs/type-system/errors.md`](specs/type-system/errors.md) — the
      name-table rows `E0109`/`E0112`, the two footnotes after it, the
      `E0212` row.
- [x] [`specs/expressions/literals.md`](specs/expressions/literals.md) — the
      interpolation requirement paragraph.
- [x] [`specs/names.md`](specs/names.md) — the trait-naming paragraph.
- [x] Drop this change's row from the [index](../README.md#index) and move
      the directory to `archive/` with status `accepted`.

## Neighbouring edits

- [ ] Cross-repository (not this repo's files): the standard library
      specification's formatting module cites `Display`/`Debug`/`Format`
      from these chapters once folded — its change belongs to that
      repository's index, and its § 1.3-style dependency list gains the item.
- [ ] Cross-repository: that repository's `modules/fmt.md` `%v`/`%s`
      contract, the `%?` verb, and its `print`-adjacent prose wait on this
      fold; nothing in *this* repository references them.
- [ ] `archive/add-declaration-attributes/` is historical — its "only
      registered target is `ToString`" text is *not* edited (archive entries
      keep their final status); the sweep below proves no active text
      disagrees.

## Sweep

- [x] `grep -rn 'ToString\|toString' --include='*.md' . | grep -v '/changes/'`
      returns only: `machine-readable.md` (a JavaScript API call, unrelated),
      and the reservation sites this change deliberately keeps — the
      prelude's reserved-names note, the traits registry bullet, the errors
      footnote (plus the `toString` references that describe the derived
      action).
- [x] `grep -rn '#Derive(ToString)' --include='*.md' . | grep -v 'archive/'`
      returns hits only inside this change's own `proposal.md`/`design.md`
      (before/after and migration text), which are excluded from folded
      chapters.
- [x] `grep -rn 'implement \`ToString\`\|without \`ToString\`\|registered targets: \`ToString\`' --include='*.md' .`
      returns nothing outside `changes/`.

## Consistency

- [x] The `str` signature reads `int | float | bool | string | Display` in
      **both** places it is spelled — the Conversion table and the chapter's
      signature index — one swap, two sites.
- [x] Every site that states the string-form requirement names `Display`:
      interpolation (traits contract, coercion row and section,
      checking-rules row and bullet, literals paragraph), `str` (contract,
      intrinsic bullets, coercion), `List.join` (composite bullet + the
      prelude "Required by" cell), `print` (prelude cell + the output
      paragraph), the derived `toString` (prelude bullet + the traits
      action bullet) — the old `ToString` spellings survive only as
      reservations and migration prose.
- [x] The derived `toString` has a resolution rule, not just an existence
      claim: the delta adds a paragraph to `Member calls` (found when
      neither step finds a candidate; yields to inherent and declared
      members; grants `T: Display` bodies the member), and design's
      dispatch bullet matches it. `u.toString()` must not be reachable only
      through the contract bullet.
- [x] The enum fallback (no impl → the table's enum row, never `E0801`) is
      stated the same way in all four places: the traits derive rules, the
      design's field-rules items, the print precedence table, and the
      errors footnote (which now names only a `struct` for `E0801`).
- [x] `print`'s check timing is stated as static-type-timed in the intrinsic
      delta (a type known at the call decides there — `unknown` by the
      precedence row; a type parameter decides at the call supplying the
      type argument) and justified against interpolation's definition-time
      regime in design's typing consequences.
- [x] The output table's completion rows cover the full type space — walk
      the prelude: `bool`/`string`/`int`/`float`/`number`/fixed-bit,
      `List`/`Array`/`Map`/`Set`, `unit`/`null`/`T?`, `Task`/`Range`/
      `Result`/`View`/`Error`/`Format`/`Ptr`, tuples, functions,
      then user `struct`/`enum` via the consult paragraph — and every kind
      appears in the delta's gained-rows block or an existing row. The one
      exception is `unknown`, which must appear in **no** rendering row:
      only as the precedence table's rejection row, the derive rule, and
      interpolation's existing `E0212`. The
      derive's third field rule ("renders directly") is total only if this
      walk passes.
- [x] The completion rows are additions only: no existing row's text is
      edited, and the base-type spellings (shortest float, `NaN`, raw
      strings) are quoted, not restated differently.
- [x] The derive example in `types/traits.md` (`print(s)`) and the struct
      example in `intrinsic-functions.md` (`print(Point…)`) are valid only
      with the `print` paragraph folded — they land in the same change, and
      a fold of one without the other is caught by the whole-or-nothing
      rule.
- [x] `E0111` still shows `#Derive(Display)` as its example — `Display`
      stays reserved-unregistered; only the note's target list changes to
      `Debug`.
- [x] `E0801` and `E0805` rows are untouched: both are trait-agnostic
      (`` `X` does not implement `Y` ``, `` `Y` is already implemented for
      `X` ``) and read correctly for `Debug`.
- [x] No delta names a file that does not exist, every
      `> **Delta**` blockquote names its target chapter file, and every
      blockquote lists **every** passage its delta touches — the
      `specs/types/traits.md` blockquote must name "Member calls" as well
      as the contracts section, attributes, and Derive.
- [x] The `Format` extension policy (future fields optional-typed) appears
      in the traits contract and in design's open-type section, both citing
      the composite-types initializer rule as the mechanism.
- [x] `structural positions` is defined once — with the output table in
      `specs/builtins/intrinsic-functions.md` — and the other tellings
      *reference* it rather than re-inventing a list: the traits recursion
      bullet (with link), design's field-rules recursion paragraph and
      `unknown` decision, the proposal's derive and print bullets, the
      consult paragraph, and the `unknown` precedence row.
- [x] `grep -n "two's complement" specs/` returns nothing — the fixed-bit
      row renders decimal numerals, where complement form plays no part.
- [x] `print`/`${…}` divergence for a `Debug`-only type is recorded in both
      the proposal's compatibility list and design's noted-consequence
      section.
- [x] `grep -rn 'E0801' --include='*.md' . | grep -v '/changes/'` — every
      hit outside `types/traits.md` (derive rules) and the `errors.md`
      footnote is a bound/impl use (`generics`, `functions`, `calls`,
      `checking-rules`, traits dispatch) that this change does not touch;
      `checking-rules.md`'s `E0801` references are all bound positions, so
      its delta needs no `E0801` edit.
- [x] The three-row precedence table and the consult paragraph in
      `specs/builtins/intrinsic-functions.md` state the same outcomes, and
      the proposal's acceptance matrix has one row per site the deltas
      touch.
- [x] `E0212`'s two conditional notes (at `print`; when the type implements
      `Debug`) appear in the errors delta and design's diagnostics table;
      the message shape itself is unchanged, and the notes are disjoint.
- [x] The traits delta's unused-parameter statement cites the note under the
      warnings table of `type-system/errors.md` ("never reported as unused"
      for a parameter) — checked against that chapter.
