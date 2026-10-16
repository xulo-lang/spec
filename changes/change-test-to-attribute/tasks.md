# Tasks: change-test-to-attribute

## Preflight

- [ ] [add-declaration-attributes](../add-declaration-attributes/proposal.md)
      is accepted and folded first. This change's deltas are written against
      the post-attribute chapters; at acceptance, re-apply them on top of the
      folded text (the grammar and traits deltas state the post-attribute
      wording they edit).
- [ ] Confirm the keyword count: the attribute change does not touch the
      keyword set; this change carries it from 42 to 41 on its own.

## Fold

- [ ] Replace `testing.md` with the full chapter in
      [`specs/testing.md`](specs/testing.md).
- [ ] Apply [`specs/grammar.md`](specs/grammar.md): remove `"test"` from the
      `Keyword` production; reword the `test` sentence in the Declarations
      opening paragraph; replace the `TestDecl` production and its prose.
- [ ] Apply [`specs/lexical-structure.md`](specs/lexical-structure.md): count
      42 → 41; remove the `test` row; add the "not reserved" sentence.
- [ ] Apply [`specs/type-system/errors.md`](specs/type-system/errors.md):
      reword the `E0107` and `E0501` rows (message shapes unchanged).
- [ ] Apply [`specs/type-system/checking-rules.md`](specs/type-system/checking-rules.md):
      reword the one bullet.
- [ ] Apply [`specs/modules/source-files.md`](specs/modules/source-files.md):
      Test row of the forms table; forward-references bullet.
- [ ] Apply [`specs/types/traits.md`](specs/types/traits.md): extend the
      placement bullet ("and on the test form (`#Test`)"); add the
      `### Test` registry entry after `### Derive`.

## Neighbouring edits

- [ ] `introduction.md` (~line 140): "the `test fn` form and the `assert`
      intrinsic" → "the `#Test` form and the `assert` intrinsic".
- [ ] `expressions/async-expressions.md` (two places): "a test function
      marked `async`" / "in a `test` function that is not marked `async`" →
      "`#Test` function".
- [ ] `builtins/intrinsic-functions.md` (~line 122): "how a `test` body
      states a requirement" → "how a `#Test` body states a requirement".
- [ ] `ast.md`: no change — `TestDeclaration` keeps its name, fields, and
      the prose at line 159 ("a test is never part of an interface" is
      ordinary English, not the keyword).

## Sweep

- [ ] `grep -rn 'test fn\|test async fn' --include='*.md' .` returns nothing
      outside `changes/` (archived and draft changes keep their old
      spellings as historical text).
- [ ] `grep -rn '"test"' grammar.md lexical-structure.md` returns nothing.
- [ ] `grep -rn '\btest\b' testing.md` — every remaining `test` is ordinary
      English prose ("a test", "the test's identity"), never the keyword
      spelling.
- [ ] Keyword consistency: lexical table count sentence says 41; the
      `Keyword` production in `grammar.md` lists 41 alternatives; no `test`
      row remains.

## Consistency

- [ ] `TestDecl` production identical in `grammar.md` and `testing.md`:
      `"#Test" [ "async" ] "fn" Identifier "(" ")" Block`.
- [ ] `E0107` carries exactly one message shape across `testing.md`,
      `errors.md`, `checking-rules.md`; the `#Test` spelling appears in its
      trigger and example.
- [ ] `E0501`'s trigger mentions `#Test`; the message shape is unchanged.
- [ ] `E0109`/`E0110` Test instances (`specs/types/traits.md`) match the
      message patterns of the `E0109`/`E0110` rows added by
      add-declaration-attributes.
- [ ] Every internal link in the folded `testing.md` resolves (run the link
      checker).

## Acceptance

- [ ] All boxes above checked; archive the directory as `accepted`; drop the
      index entry from [`../changes/README.md`](../README.md).
