# Grammar — Delta: Closed Range Operator

> **Delta** — replaces the `SymbolOperator` production, the `RangeExpression`
> production, the spread/range paragraph, and the `RangePattern` production in
> `grammar.md`.

## Operators and delimiters

`SymbolOperator` becomes:

```text
SymbolOperator =
      "." | "?" | "!" | "=" | "==" | "!=" | "<" | ">" | "<=" | ">="
    | "+" | "-" | "*" | "/" | "%" | "**" | "<<" | ">>" | "&" | "|" | "^" | "~"
    | "=>" | "..<" | "..=" | "..." | "?." | "??" | "@" | "$" | "::" | ":=" | "#" ;
```

## Expressions

`RangeExpression` becomes:

```text
RangeExpression      = BitOrExpression [ ( "..<" | "..=" ) BitOrExpression ] ;
```

The paragraph that follows the expression grammar becomes:

The closed range and the spread use distinct tokens: `..=` is the infix
closed-range operator, while `...` is a prefix that an element of a list
literal or a brace map MAY begin with. `[...xs]` spreads `xs`, while
`[1..=5]` is one closed-range element
([lexical-structure.md](lexical-structure.md)).

## Patterns

`RangePattern` becomes:

```text
RangePattern   = ( IntegerLiteral | FloatLiteral ) ( "..<" | "..=" )
                                 ( IntegerLiteral | FloatLiteral ) ;
```
