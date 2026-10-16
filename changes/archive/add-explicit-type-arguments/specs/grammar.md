# Grammar — Delta: Explicit Type Arguments

> **Delta** — in `grammar.md`: replaces the `PostfixOperand` production of Expressions, adds the `CallTypeArgs` rule to Expressions, replaces the `ComponentCall` production of Components and UI, and adds `CallTypeArgs` to the Expressions row of the nonterminals index.

## Expressions — `PostfixOperand`

```text
PostfixOperand       = [ CallTypeArgs ] "(" [ ArgumentList ] ")"
                     | "[" Expression "]"
                     | "." ( Identifier | IntegerLiteral )
                     | "?." ( Identifier | IntegerLiteral )
                     | "?" ;
CallTypeArgs         = "<" [ TypeList ] ">" ;
```

## Expressions — `CallTypeArgs`

Added after the paragraph that begins `` `?` follows an expression in two roles ``:

`CallTypeArgs` is recognized only in call position, and only when the
expression before it ends in an identifier — a bare identifier or a member
access — so that a callee reached through a subscript, a call, or a closing
bracket takes no list. Whether a `<` after such a callee starts a
`CallTypeArgs` is decided by two conditions:

1. the material between that `<` and the matching `>` parses as a
   comma-separated list of types — the empty material counts as a zero-length
   list, so `f<>(…)` reaches the arity diagnostic instead of falling back to a
   comparison; and
2. the token immediately after the matching `>` is `(`.

If either condition fails, the `<` is the relational operator and the
expression is a comparison. The scan for the matching `>` counts angle depth
and skips over `(` `)`, `[` `]`, and `{` `}` groups; it MUST NOT consult name
resolution, and backtracking from the type-argument reading to the comparison
reading is allowed whenever the call parse does not complete. Under the rule
`x = a < b > (c)` reads as the call `a<b>(c)`, and an expression that means two
comparisons is written `f((a < b), c > (d))`, where the material after `<` no
longer parses as a type list.

`TypeArgs`, the argument list of a named type, requires at least one element
and is unchanged; `CallTypeArgs` is a separate production that admits the
empty list, because a zero-length list at a call must reach the arity
diagnostic rather than fall back to a comparison. `>>` and `>=` are single
tokens and never match, so a nested type argument keeps the spacing the
nested-bracket rule already requires: `first<Map<String, Int> >(m)` is
well-formed and `first<Map<String, Int>>(m)` is not, the latter failing to
parse with a note to insert the space. Typing and diagnostics are in
[expressions/calls.md](expressions/calls.md).

## Components and UI — `ComponentCall`

```text
ComponentCall  = Identifier [ CallTypeArgs ] "(" [ ArgumentList ] ")" ComponentBlock
               | Identifier ComponentBlock ;
```

`ComponentBlock` and `ChildItem` are unchanged. An invocation written with an
argument list and no block — `Text("Hi")`, `first<Int>([1, 2, 3])` — parses as
an ordinary call through `PostfixOperand`, and the two read the same.

## Nonterminals index

In the Expressions row that ends with `PostfixOperand`, `CallTypeArgs` follows
`PostfixOperand`.
