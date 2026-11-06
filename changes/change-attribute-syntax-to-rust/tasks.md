# Tasks — change-attribute-syntax-to-rust

## Acceptance

- [ ] Fold the deltas under [`specs/`](specs/) into `grammar.md`,
      `lexical-structure.md`, `ast.md`, `introduction.md`,
      `documentation-comments.md`, `modules/source-files.md`,
      `type-system/checking-rules.md`, `type-system/errors.md`,
      `builtins/prelude.md`, `builtins/intrinsic-functions.md`,
      `expressions/async-expressions.md`, `types/traits.md`, and
      `testing.md`. The old `#Name` spelling and the old capitalized
      built-in names must not survive outside historical change records.
- [ ] Rebase `changes/add-macro-system` onto this spelling, merging its macro
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
      - Keep the title rule special after the widening: `#[test("…")]` outside
        a full `TestDecl` remains a syntax error, not an `E0109`/`E0110`
        ordinary attribute. The widened `AttributeArg` MUST NOT reinterpret
        the test title as a string argument of a generic attribute.
      - Re-spell `#Macro` → `#[macro]`, `#Derive` → `#[derive]`, `#Test` →
        `#[test]`, every user-attribute example, and the prose in its
        `proposal.md`, `design.md`, and `specs/`.
- [ ] Update `changes/README.md`: keep this change's draft row and, once
      accepted, replace `add-macro-system`'s “`#Macro`” wording with
      “`#[macro]`”.
- [ ] Remove this draft row from `changes/README.md`, add an `accepted`
      archive row, and move the directory to `archive/`.

## Cross-references

- [ ] Check `documentation-comments.md`: attribute lines such as `#Test` are
      now `#[test]`, and the whitespace/comment rules that mention an
      attribute line must use the new spelling.
- [ ] Check `types/traits.md` and `testing.md` for the test shape message:
      it MUST not quote `#Test` or `#[Test]`, and it MUST use `test`.
- [ ] Check `type-system/errors.md` for every `#` attribute example, including
      the gallery and the `E0107`–`E0112`, `E0212`, and `E0501` rows. Messages
      name the lowercase attribute; derive targets keep their case.
- [ ] Check `grammar.md`'s production index for `AttributeName` and check
      the `AttributeNameNode` definition in `ast.md`; no other node
      shape changes.
- [ ] Check every `specs/` file's first heading and `> **Delta** — …`
      blockquote against the `changes/README.md` convention.

## Tests

- [ ] Parser tests: `#[derive(Debug)]`, `#[test]`, `#[test("parse test")]`,
      `#[test()]`, `#[macro(...)]`, and a user attribute all parse;
      `#[test]` with parameters, a return type, generics, a `where` clause, or
      `pub` is `E0110`.
- [ ] Parser tests: `#Derive`, `#Test`, `#Test("parse test")`, and `#Inline`
      are rejected as syntax errors; a bare `#` is rejected.
- [ ] Reserved-word tests: `derive` and `macro` remain illegal as identifiers
      (`fn derive()`, `let macro = 1`), but are accepted as `#[derive]` and
      `#[macro]`; `#[test]` uses the ordinary identifier `test`.
- [ ] AST tests: an attribute's span covers `#` through `]`;
      `AttributeNameNode.name` is the lowercase string; the `TestDecl` title
      field is `null` for `#[test]` and `#[test()]`, and is the string for
      `#[test("…")]`.
- [ ] Diagnostic tests: `E0107`–`E0112` examples and messages use lowercase
      attribute names; `E0110`'s test-form message names `#[test]` /
      `#[test("…")]`.
- [ ] Update the external grammar snapshots (`xulo.ebnf`, `xulo.pest`,
      tree-sitter files) in the implementation repository after the grammar
      delta is folded.
- [ ] Run the implementation repository's parser and error-message suites
      after the rebase; no old `#Name` or `#[Derive]` fixture should remain.
