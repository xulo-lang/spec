# Grammar — Delta: Macro Syntax

> **Delta** — replaces the attribute productions in the Declarations
> section, adds `AttributeList` to `TypeAlias`, `LetDecl`, and `ConstDecl`,
> replaces the declaration-list, attribute-argument, and `#[test(...)]`
> paragraphs, adds `MacroCall` and `QuoteCall` to `PrimaryExpression`, and
> adds the new nonterminals to the production index of `grammar.md`. The rest
> of the grammar is unchanged.

## Declarations

Replace the attribute productions with:

```text
Attribute      = "#" "[" AttributeName [ "(" [ AttributeArgs ] ")" ] "]" ;
AttributeName  = Identifier | "derive" | "macro" ;
AttributeArgs  = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg   = [ Identifier "=" ] AttributeValue ;
AttributeValue = Identifier | StringLiteral | IntegerLiteral
               | FloatLiteral | BooleanLiteral ;
AttributeList  = { Attribute } ;
```

A bare identifier is the common form — `#[derive(Debug)]` — and `name = value`
is the form a macro definition uses — `#[macro(kind = "derive", target =
Serialize)]`. For a built-in attribute, the registry entry decides the shape:
an attribute that requires identifiers rejects a value (`E0110`), and a macro
whose `#[macro]` metadata has the wrong keys or kinds is `E1010`. A user
attribute macro receives the values and validates them itself.

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
A `#[macro]` definition is an ordinary `FnDecl` carrying the `#[macro]` attribute;
it adds no declaration production.

### Replaced prose

Replace the paragraph that begins “The five item declarations” with:

> The eight item declarations — `FnDecl`, `StructDecl`, `EnumDecl`,
> `TraitDecl`, `ImplDecl`, `TypeAlias`, `LetDecl`, and `ConstDecl` — MAY carry
> an attribute list, written outermost, before `pub` and `async`; which
> attributes are defined, and on which of them, is a checking rule and not a
> grammatical one ([traits.md](types/traits.md)); the two extern forms carry
> no attribute list.

Replace the paragraph that begins “An attribute is written” with:

> An attribute is written `#[name]` or `#[name(args)]`. The built-in attribute
> names are lowercase. `derive` and `macro` are reserved words accepted only in
> the attribute-name position; `test` is an ordinary identifier. Attribute
> names occupy their own namespace and bind nothing. An argument is a
> positional value or a `name = value` entry; the registry entry decides which
> shape a built-in attribute accepts (`E0110`), and a user attribute macro
> receives the values and validates them itself. The declarations that carry
> an `AttributeList` are `FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`,
> `ImplDecl`, `TypeAlias`, `LetDecl`, and `ConstDecl`.

Replace the paragraph that begins “When the full test form does not match”
with:

> When the full test form does not match, `#[test]` and `#[test(...)]` are
> ordinary attributes on the `FnDecl`. A bare `#[test]` before a `fn` with
> parameters, a return type, generic parameters, a `where` clause, or `pub` is
> `E0110`; so are `#[test()]`, `#[test(name, …)]`, `#[test("…")]`, and
> `#[test(1)]`, because the test attribute takes no arguments. Every
> `#[test(...)]` group admitted by the widened `AttributeArg` grammar parses as
> an ordinary attribute and is `E0110`; there is no syntax-error fallback for a
> well-formed argument. The registry rules of [traits.md](types/traits.md) give
> the message for the `E0110` case
> (`` attribute `test` takes no arguments and is written `#[test]`, followed by
> `[async] fn name() { … }` ``).

**Merge dependency.** The `#[test(...)]` paragraph above is the base text,
where `#[test]` takes no arguments. `add-test-options` replaces the
`TestDecl`/`TestOptions` productions and makes valid option groups legal.
After the merge, `E0110` applies only to a `#[test(...)]` group that is not a
valid `TestOptions` group; a positional value is still `E0110`, not a syntax
error, and a valid option group is accepted. Do not fold this paragraph
without that adjustment.

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
