# Tasks — change-attribute-spelling

## Acceptance

- [x] Fold the deltas under [`specs/`](specs/) into `grammar.md`,
      `lexical-structure.md`, `ast.md`, `introduction.md`,
      `documentation-comments.md`, `modules/source-files.md`,
      `type-system/checking-rules.md`, `type-system/errors.md`,
      `builtins/prelude.md`, `builtins/intrinsic-functions.md`,
      `expressions/async-expressions.md`, `types/traits.md`, and
      `testing.md`. The old `#Name` spelling and the old capitalized
      built-in names must not survive outside historical change records.
- [x] Rebase `changes/add-macro-system` onto this spelling, merging its macro
      deltas rather than overwriting them:
      - The grammar keeps this change's
        `Attribute = "#" "[" AttributeName [ "(" [ AttributeArgs ] ")" ] "]" ;`
        and `AttributeName = Identifier | "derive" | "macro" ;`, while retaining
        `add-macro-system`'s widened
        `AttributeArg = [ Identifier "=" ] AttributeValue ;` and its
        `AttributeValue` alternatives. This change's “`AttributeArg` unchanged”
        note is true against the main chapters only; the rebase must not
        narrow the macro argument grammar back to `Identifier`.
      - The AST node is
        `AttributeNode { name: AttributeNameNode; args: AttributeArgNode[] |
        null }`: this change supplies `name`, `add-macro-system` supplies
        `args` and `AttributeArgNode`.
      - Do not restore a test title: `#[test]` takes no arguments, so
        `#[test()]`, `#[test("…")]`, and `#[test(name, …)]` outside a full
        `TestDecl` are ordinary attributes and `E0110`; the widened
        `AttributeArg` must not create a title path.
      - Re-spell `#Macro` → `#[macro]`, `#Derive` → `#[derive]`, `#Test` →
        `#[test]`, every user-attribute example, and the prose in its
        `proposal.md`, `design.md`, and `specs/`. Drop its test-title
        references, including the fixed string-title form and the `#Test()`
        example; `#[test]` takes no arguments and `TestDeclaration` loses
        `title`.
- [x] Update `changes/README.md`: keep this change's draft row and, once
      accepted, replace `add-macro-system`'s “`#Macro`” wording with
      “`#[macro]`”.
- [x] Remove this draft row from `changes/README.md`, add an `accepted`
      archive row, and move the directory to `archive/`.

## Cross-references

- [x] Check `documentation-comments.md`: attribute lines such as `#Test` are
      now `#[test]`, and the whitespace/comment rules that mention an
      attribute line must use the new spelling.
- [x] Check `types/traits.md` and `testing.md` for the test shape message:
      it MUST not quote `#Test` or `#[Test]`, and it MUST use `test`.
- [x] Check `type-system/errors.md` for every `#` attribute example, including
      the gallery and the `E0107`–`E0112`, `E0212`, and `E0501` rows. Messages
      name the lowercase attribute; derive targets keep their case.
- [x] Check `grammar.md`'s production index for `AttributeName` and check
      the `AttributeNameNode` definition in `ast.md`; no other node
      shape changes.
- [x] Check every `specs/` file's first heading and `> **Delta** — …`
      blockquote against the `changes/README.md` convention.

## Tests

Implementation follow-ups; not run in this specification repository.

- [ ] Parser tests: `#[derive(Debug)]`, `#[test]`, `#[macro(...)]`, and a
      user attribute all parse; `#[test]` with parameters, a return type,
      generics, a `where` clause, or `pub` is `E0110`; `#[test()]`,
      `#[test("parse test")]`, and `#[test(name)]` are not `TestDecl`s and are
      `E0110` when their argument group parses.
- [ ] Parser tests: `#Derive`, `#Test`, `#Test("parse test")`, and `#Inline`
      are rejected as syntax errors; a bare `#` is rejected.
- [ ] Reserved-word tests: `derive` and `macro` remain illegal as identifiers
      (`fn derive()`, `let macro = 1`), but are accepted as `#[derive]` and
      `#[macro]`; `#[test]` uses the ordinary identifier `test`.
- [ ] AST tests: an attribute's span covers `#` through `]`;
      `AttributeNameNode.name` is the lowercase string; `TestDeclaration` has
      no `title` field.
- [ ] Diagnostic tests: `E0107`–`E0112` examples and messages use lowercase
      attribute names; `E0110`'s test-form message names `#[test]` and says
      the attribute takes no arguments.
- [ ] Update the external grammar snapshots (`xulo.ebnf`, `xulo.pest`,
      tree-sitter files) in the implementation repository after the grammar
      delta is folded.
- [ ] Run the implementation repository's parser and error-message suites
      after the rebase; no old `#Name` or `#[Derive]` fixture should remain.
