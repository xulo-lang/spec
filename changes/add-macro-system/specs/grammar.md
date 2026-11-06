# Grammar — Delta: Macro Syntax

> **Delta** — replaces the attribute productions in the Declarations
> section, adds `AttributeList` to `TypeAlias`, `LetDecl`, and `ConstDecl`,
> adds `MacroCall` and `QuoteCall` to `PrimaryExpression`, and adds the new
> nonterminals to the production index of `grammar.md`. The rest of the
> grammar is unchanged.

## Declarations

Replace the attribute productions with:

```text
Attribute      = "#" Identifier [ "(" [ AttributeArgs ] ")" ] ;
AttributeArgs  = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg   = [ Identifier "=" ] AttributeValue ;
AttributeValue = Identifier | StringLiteral | IntegerLiteral
               | FloatLiteral | BooleanLiteral ;
AttributeList  = { Attribute } ;
```

A bare identifier is the common form — `#Derive(Debug)` — and `name = value`
is the form a macro definition uses — `#Macro(kind = "derive", target =
Serialize)`. The registry entry decides the shape: an attribute that requires
identifiers rejects a value (`E0110`), and a macro whose `#Macro` metadata has
the wrong keys or kinds is `E1010`.

Replace the three declaration productions that gain an attribute list:

```text
TypeAlias = [ AttributeList ] [ "pub" ] "type" Identifier GenericParams?
            "=" Type ;
LetDecl   = [ AttributeList ] [ "pub" ] "let" ( MutableBinding | SimpleBinding
                                              | DestructuringBinding ) ;
ConstDecl = [ AttributeList ] [ "pub" ] "const" Identifier [ ":" Type ]
            "=" Expression ;
```

`AttributeList` now precedes the declaration on every attribute host:
`FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`, `ImplDecl`, `TypeAlias`,
`LetDecl`, and `ConstDecl`. A local `let` uses the same `LetDecl` production,
so it may carry an attribute list syntactically; whether the attribute macro
accepts a `let` host is the macro's `ast::AttributeInput.target.kind` decision.
A `#Macro` definition is an ordinary `FnDecl` carrying the `#Macro` attribute;
it adds no declaration production.

## Expressions

Replace `PrimaryExpression` and add the two macro forms:

```text
PrimaryExpression = Literal | TemplateLiteral | Identifier
                  | MacroCall | QuoteCall | PathExpr
                  | "(" Expression ")" | TupleLiteral
                  | ListLiteral | MapLiteral
                  | IfExpression | MatchExpression | PanicExpression
                  | ClosureExpression | ComponentCall
                  | SpawnExpression | LockExpression ;

MacroCall         = Path "!" "(" [ MacroArgumentList ] ")" ;
MacroArgumentList = Expression { "," Expression } [ "," ] ;

QuoteCall         = "ast" "::" "quote" "!" "{" { QuotePiece } "}" ;
QuotePiece        = QuoteHole
                  | ? any balanced token sequence that does not begin with
                      the two characters "${" and does not contain an
                      unmatched "}" ? ;
QuoteHole         = "${" Expression "}" ;
```

`MacroCall` takes expressions only: no labels, no `$` bindings, and no
trailing-argument rules beyond the comma list. A function macro receives each
argument as an `ast::Value`: a literal becomes a scalar value and every other
expression becomes an `ast::AstNode`.

`QuoteCall` is the one brace-delimited macro form and is spelled exactly
`ast::quote! { … }`. Its body is not an ordinary `Block`: it may contain
declarations, statements, and partial fragments — the pieces are balanced
tokens — and `${ Expression }` marks a hole. Nested braces, parentheses, and
brackets inside a `QuotePiece` are balanced and belong to the piece; a `}`
that closes the quote ends the body. The compiler substitutes the holes and
parses the result as a sequence of declarations and statements.

**Ambiguity notes.** No macro form existed before this change, so every
position that now begins one was previously a syntax error: `!` is only a
prefix operator, so `Path "!"` cannot be confused with a binary operator, and
`ast::quote!` is a path followed by `!` and `{`, not a path followed by a
call.

The parser decides a `Path` followed by `!` by one token of lookahead:

- `Path` `!` `(` is a `MacroCall`;
- the path `ast::quote` followed by `!` `{` is a `QuoteCall`;
- any other `Path` `!` is a syntax error.

`QuoteCall` has priority over `MacroCall`: its head is the literal path
`ast::quote` and its delimiter is `{`, while `MacroCall` requires `(`. The two
therefore never overlap. Whitespace between the path, `!`, and the delimiter
does not change the choice; `a !(b)` is a `MacroCall` exactly as `a!(b)` is.
Because `!=` is one token, `a != b` never begins a macro form. The prefix
`!x` is likewise unaffected, since it has no `Path` before the `!`.

## Production index

Add these nonterminals to the Declarations row of the index:

```text
AttributeArg, AttributeValue
```

`AttributeArg` already appears in the index; its shape is widened.
`AttributeValue` joins it. Add these nonterminals to the Expressions row:

```text
MacroCall, MacroArgumentList, QuoteCall, QuotePiece, QuoteHole
```
