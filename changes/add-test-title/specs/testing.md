# Tests — Delta: Test Title

> **Delta** — replaces the **Declaration** section of `testing.md` and adds
> one sentence to **The name**.

## Declaration

The form is `TestDecl` in the grammar ([grammar.md](grammar.md)):

```text
TestDecl = "#Test" [ "(" [ StringLiteral ] ")" ] [ "async" ] "fn"
           Identifier "(" ")" Block ;
```

- A `#Test` function appears at file scope, among a module's top-level
  declarations ([modules/source-files.md](modules/source-files.md)); it is
  not a statement and appears nowhere else.
- It carries no `pub`, no parameters, no return type, no generic parameters,
  and no `where` clause: the body is an implicit `fn(): Unit`, and the
  declaration is never part of an interface
  ([modules/visibility.md](modules/visibility.md)).
- The identifier is the test's identity as a tool reports it. It need not be
  unique across a module — two tests MAY share a name, and a tool tells them
  apart by their positions or their titles — but it is never a name in the
  value namespace ([The name](#the-name) below).
- An optional parenthesized string between the attribute and the signature —
  `#Test("parse test")` — is the test's **title**: metadata the declaration
  carries for a tool's reporting. It is not a name in any namespace, it
  binds nothing, and it need not be unique. An empty group `#Test()` and the
  bare form both mean no title; the bare form is **RECOMMENDED** when no
  title is wanted.
- The optional `async` marker says the body suspends; it is spelled exactly
  as `async` is spelled on a function, and it follows the attribute and its
  optional title group (`#Test("…") async fn`, never `async #Test fn`).

```xulo
#Test("parse test")
fn testParse() {
  assert(parseOk(), "parses")
}

#Test
fn mapGetAnswersNull() {
  let m = { a: 1 }
  assert(map_get(m, "z") == null, "absent key is null")
}

#Test()
async fn delayedTaskFinishes() {
  await Time::sleep(1)
}
```

The examples write `#Test` on its own line directly above `fn`, and that
placement is **RECOMMENDED**: the attribute marks the whole declaration and
reads better apart from the signature. The grammar is newline-insensitive
([lexical-structure.md](lexical-structure.md)), so `#Test fn name() { … }` on
one line is equally well-formed.

## The name — added sentence

After the first bullet of the identifier rule:

> The title, when written, is not a name either — it lives in metadata
> position and cannot be referenced; every rule above is about the
> identifier.
