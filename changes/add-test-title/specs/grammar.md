# Grammar — Delta: Test Title

> **Delta** — replaces the `TestDecl` production and its two prose paragraphs
> in the Declarations section of `grammar.md`.

## TestDecl

The production replaces the current one:

```text
TestDecl = "#Test" [ "(" [ StringLiteral ] ")" ] [ "async" ] "fn"
           Identifier "(" ")" Block ;
```

The prose paragraphs replace the current ones:

A `TestDecl` is a function declaration with a fixed shape: `#Test` precedes
its optional title group, `async`, and `fn` (never `async #Test fn`), the
parameter list and the return type are absent — the body is an implicit
`fn(): Unit` — and it carries no `pub`, no generic parameters, and no
`where` clause. The identifier is the test's identity as a tool reports it,
not a name in the value namespace: referencing it is a compile-time error
(`E0107`). The optional `async` marker makes the body an `async` context
([async-expressions.md](expressions/async-expressions.md)); the declaration's
checking, inertness, and pass/fail criterion are specified in
[testing.md](testing.md).

The head token is written exactly `#Test`; the production carries the
attribute name as a literal, so the test form accepts no other attributes,
and its optional parenthesized group carries a `StringLiteral` title — an
absent group and an empty group `()` both mean no title. When the token
sequence `#Test` `(" StringLiteral? ")")? `async`? `fn` Identifier `(` `)`
Block appears at file scope, the declaration is a `TestDecl` and no
attribute lookup runs. A `#Test` whose group holds an identifier parses as
an ordinary `FnDecl` carrying the attribute, and the registry rules of
[traits.md](types/traits.md) diagnose it (`E0110` —
`` attribute `Test` takes the form `#Test ("…")? [async] fn name() { … }` ``);
a group holding anything else — a numeral, for instance — is a syntax error.
A `#Test` that precedes a `fn` with parameters, a return type, generic
parameters, a `where` clause, or `pub` does not match `TestDecl` and is the
same `E0110`. Written on a `struct`, `enum`, `trait`, or `impl`, `#Test` is
`E0109`.
