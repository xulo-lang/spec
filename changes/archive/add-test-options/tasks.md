# Tasks: add-test-options

## Preflight

- [x] The `add-macro-system` change is accepted: its widened `AttributeArg`,
      `AttributeValue`, `AttributeArgNode`, and `AttributeValueNode` are in the
      chapters. The deltas below are rebased onto that grammar and AST and
      must not redefine or narrow them.

## Fold

- [x] Replace `TestDecl`, add `TestOptions`, `TestOption`, and
      `TestExpectation`, and replace the `TestDecl` prose in
      [`specs/grammar.md`](specs/grammar.md); add the new productions to the
      declarations production index. `AttributeArg`, `AttributeValue`, and the
      attribute-argument paragraph are already in the accepted chapters.
- [x] Add `TestMode`, `mode`, `expectedMessage`, `isSkipped`, `skipReason`,
      and `timeoutMs` to `TestDeclaration`, and replace the `TestDeclaration`
      note in [`specs/ast.md`](specs/ast.md). `AttributeNode`,
      `AttributeArgNode`, and `AttributeValueNode` are already in the accepted
      chapters.
- [x] Replace the declaration grammar and bullet, replace **Expected failure**
      with **Options**, and replace the **Running** criterion in
      [`specs/testing.md`](specs/testing.md).
- [x] Replace the `### Test` entry in
      [`specs/types/traits.md`](specs/types/traits.md).

## Cross-references

- [x] `grammar.md` — `TestOptions`, `TestOption`, and `TestExpectation` are
      indexed; the fallback paragraph names the new accepted forms and treats
      a positional value as `E0110` rather than a syntax error.
- [x] `ast.md` — confirm the accepted `AttributeNode.args` carries
      `AttributeArgNode`s with optional names and value shapes;
      `TestDeclaration` has no `attributes` or `title`; the mode, expectation,
      skip, and timeout fields are documented; `TestMode` is exported like the
      other AST type aliases.
- [x] `testing.md` — normal, `should_fail`, and `should_panic` are defined by
      distinct outcomes; cancellation passes `should_fail` and fails
      `should_panic`; timeout is a non-panic failure; skip prevents the run and
      has no outcome; message matching is a substring of the failure message.
- [x] `types/traits.md` — the registry message for a malformed test group uses
      the new forms; `E0109` and `E0107` are unchanged; deferred `category`,
      `focus`, `retry`, and `threads` are `E0110`.
- [x] `type-system/errors.md` — no new code; check that `E0110`'s generic
      trigger still covers a malformed option group.
- [x] `changes/README.md` — keep the draft row with the expectation, skip, and
      timeout forms and the deferred-option note.
- [x] `changes/add-macro-system` — the merge requirement is recorded and that
      change is accepted; keep this change rebased onto its grammar and AST.
- [x] `testing.md` — the “everything else” paragraph no longer claims that
      selection and execution are entirely a tool concern, because `skip` is
      specified here.

## Tests

- [ ] Parser tests: `#[test]`, `#[test(should_fail)]`,
      `#[test(should_panic)]`, `#[test(should_panic = "msg")]`,
      `#[test(skip)]`, `#[test(skip = "reason")]`, `#[test(timeout = 100)]`,
      and `#[test(should_panic, timeout = 100)]` form `TestDecl`.
- [ ] Parser tests: `#[test()]`, `#[test(should_fail = "msg")]`,
      `#[test(should_panic, should_fail)]`, `#[test(skip, timeout = 100)]`,
      `#[test(timeout = 0)]`, `#[test(timeout = "100")]`,
      `#[test(category = "slow")]`, and `#[test(focus)]` are `E0110` when
      they parse as attributes; `#[test("msg")]`, `#[test(1)]`, and
      `#[test(timeout = 1.5)]` also parse as attributes and are `E0110`;
      `#[test(-1)]` remains a syntax error.
- [ ] AST tests: a bare identifier argument and a named string/integer
      argument are distinguishable through `AttributeArgNode.name` and
      `value`; `mode` is `normal`, `shouldFail`, or `shouldPanic`;
      `expectedMessage` is non-null only for `should_panic = "msg"`;
      `isSkipped` and `skipReason` cover no skip, bare skip, and skip with a
      reason; `timeoutMs` is a positive integer `NumberLiteral` or null.
- [ ] Diagnostic tests: a malformed option group on a non-matching `fn` is
      `E0110` with the new message shape; on a `struct`, `enum`, `trait`, or
      `impl` it is `E0109`; `E0107` is unchanged.
- [ ] Semantic tests: a normal test fails on panic, cancellation, and timeout;
      `should_fail` passes on panic, runtime failure, cancellation, and
      timeout; `should_panic` passes on panic and runtime failure but fails on
      cancellation and timeout; a message match is a case-sensitive substring.
- [ ] Skip tests: a skipped test is not executed, is reported as skipped, and
      is not counted as a pass or failure; its body is still checked.
- [ ] Timeout tests: a test that settles before the deadline keeps its normal
      outcome; one that does not is `timedOut`; an `async` timeout cancels the
      task but reports `timedOut`, not `cancelled`.
- [ ] Rebase test: confirm the accepted `add-macro-system` widened
      `AttributeArg`/`AttributeValue` and `AttributeArgNode`/
      `AttributeValueNode` are not narrowed by this change, and that test
      options are a separate `TestOptions` production.
