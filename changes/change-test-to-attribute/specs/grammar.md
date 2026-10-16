# Grammar — Delta: `#Test` Test Declaration

> **Delta** — edits the `Keyword` production and the Declarations section's
> opening paragraph, and replaces the `TestDecl` production and its prose in
> `grammar.md`.

## Keywords

The `Keyword` production loses `test` (41 keywords remain); the row's other
alternatives are unchanged:

```text
        | "super" | "trait" | "true" | "type" | "use" | "where"
```

## Declarations — opening paragraph, edited sentences

The opening paragraph of the Declarations section keeps its structure; two
sentences change. The paragraph begins (unchanged):

> Every alternative of `Declaration` MAY carry `pub`, except `impl` and
> `TestDecl`: `impl` introduces no name and is never part of an interface,
> and a `#Test` declaration is never part of an interface either
> ([visibility.md](modules/visibility.md)); the four component declarations
> carry no visibility marker either
> ([state.md](components/state.md)). `async` follows `pub` and precedes `fn`.

(The `test` → `#Test` reword is the only change in this paragraph; the
remainder of the paragraph — generic parameters, `where`, `type` — is
unchanged.)

## TestDecl

The production replaces the current one:

```text
TestDecl = "#Test" [ "async" ] "fn" Identifier "(" ")" Block ;
```

The prose paragraph replaces the current one:

A `TestDecl` is a function declaration with a fixed shape: `#Test` precedes
`async` and `fn` (never `async #Test fn`), the parameter list and the return
type are absent — the body is an implicit `fn(): Unit` — and it carries no
`pub`, no generic parameters, and no `where` clause. The identifier is the
test's identity as a tool reports it, not a name in the value namespace:
referencing it is a compile-time error (`E0107`). The optional `async` marker
makes the body an `async` context
([async-expressions.md](expressions/async-expressions.md)); the declaration's
checking, inertness, and pass/fail criterion are specified in
[testing.md](testing.md).

The head token is written exactly `#Test`; the production carries the
attribute name as a literal, so the test form accepts no other attributes.
When the token sequence `#Test` `async`? `fn` Identifier `(` `)` Block
appears at file scope, the declaration is a `TestDecl` and no attribute
lookup runs. A `#Test` that precedes a `fn` with parameters, a return type,
generic parameters, a `where` clause, or `pub` does not match `TestDecl`:
the declaration parses as an ordinary `FnDecl` carrying the attribute, and
the registry rules of [traits.md](types/traits.md) diagnose it (`E0110` —
`` attribute `Test` takes the form `#Test [async] fn name() { … }` ``).
Written on a `struct`, `enum`, `trait`, or `impl`, `#Test` is `E0109`.
