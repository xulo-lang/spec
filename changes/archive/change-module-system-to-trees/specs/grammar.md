# Grammar — Delta: Header, Paths, and Keywords

> **Delta** — in `grammar.md`: replaces the **Keywords** production, the
> `Operator` note about `::`, the **Program and modules** section, the
> `NamedType` production, the `PrimaryExpression`/`VariantPath` productions,
> and the corresponding rows of the nonterminals index.

## Keywords

The language has exactly the following closed set of 42 keywords; every other
word is an ordinary identifier unless it is reserved.

```text
Keyword = "and" | "as" | "async" | "await" | "break" | "const"
        | "continue" | "copy" | "defer" | "else" | "enum" | "false" | "fn"
        | "for" | "if" | "impl" | "in" | "is" | "let"
        | "lock" | "match" | "mod" | "move" | "mut" | "not" | "null" | "or"
        | "panic" | "pub" | "return" | "self" | "shared" | "spawn" | "struct"
        | "super" | "test" | "trait" | "true" | "type" | "use" | "where"
        | "while" ;
```

A `ReservedWord` is a word whose spelling appears in the reserved list of
[lexical-structure.md](lexical-structure.md); it is neither a keyword nor an
identifier, and using one as a name is ill-formed. The reserved list excludes
every keyword above. `State`, `Store`, `Effect`, and `Environment` are
contextual: they are meaningful only directly after `@`.

## Operators and delimiters — the `::` note

This is the complete inventory. `::` is the path separator — packages,
modules, enum variants, and associated names (`lsp-server::router`,
`form::validate`, `Theme::Dark`, `Math::PI`) — and `:=` is the
mutable-binding sugar of a `let` declaration; the connectives are the word
operators `and`, `or` and the prefix `!`, and no other language's symbolic
spellings for them exist. At each position the lexer takes the longest token
in the inventory.

## Program and modules

A source file holds one module body: a header of module declarations, then
the body's declarations. All three header forms — `mod`, `use`, and
`pub use` — appear in the header, before any other top-level declaration
([source-files.md](modules/source-files.md)); a header form written in the
body is `E0605`. Visibility is written with `pub`, and the default is
private everywhere.

```text
Program      = { HeaderItem } { Declaration } [ Entry ] ;
HeaderItem   = ModDecl | UseDecl | PubUse ;
Declaration  = FnDecl | StructDecl | EnumDecl | TraitDecl | ImplDecl
             | TypeAlias | LetDecl | ConstDecl | TestDecl ;

ModDecl      = [ "pub" ] "mod" Identifier ( ModuleBody | ";" ) ;
ModuleBody   = "{" { HeaderItem } { Declaration } "}" ;
UseDecl      = "use" [ "type" ] UseTarget ";" ;
PubUse       = "pub" "use" ( UseTarget | PubUseLocal ) ";" ;

UseTarget    = UsePath | UseGroup | UseGlob ;
UsePath      = PathHead "::" Identifier { "::" Identifier } [ "as" Identifier ]
             | PathHead "as" Identifier ;
UseGroup     = PathHead "::" "{" [ UseEntry { "," UseEntry } [ "," ] ] "}" ;
UseGlob      = PathHead "::" "*" ;
UseEntry     = Identifier [ "as" Identifier ] ;
PubUseLocal  = "{" [ Identifier { "," Identifier } [ "," ] ] "}" ;

PathHead     = PackageName | Identifier | "self" | "super" ;
Path         = PathHead { "::" Identifier } ;
PackageName  = Identifier { "-" Identifier } ;
```

`UsePath` admits a head-only target only in its second alternative, where
`as` renames the namespace binding; every other target carries at least one
segment after its head. `PubUseLocal` re-exports names already in this
module's scope and takes no `as`: the name is the export. An inline `mod`
body reuses `{ HeaderItem } { Declaration }`, so a nested module has its own
header and the header rule does not change with depth.

`Entry` is the entry point of an executable program — a module-level `main`
with the signature [source-files.md](modules/source-files.md) allows (`fn
main()`, `fn main(): View`, or `async fn main()`, no parameters), declared in
the package's root module. Since `FnDecl` derives `main` too, the grammar
constrains neither its form nor its position; `Entry` names it in the
top-level shape of `Program`.

```text
Entry      = EntryPoint ;
EntryPoint = [ "pub" ] [ "async" ] "fn" "main" "(" ")" [ ":" Type ] Block ;
```

**Ambiguity note.** The header forms begin only with `mod`, `use`, or `pub`
followed by `use`, and end with `;` (or, for an inline `mod`, with its own
block), so `{ HeaderItem } { Declaration }` splits at the first token outside
that set — a `pub` that begins a declaration continues the split by its
follower (`pub use` stays in the header; `pub fn` and the rest start the
body). `UseTarget` alternatives are distinguished by what follows the
head: `as`, `::`, or nothing (which no alternative allows). In an expression
or a type, a hyphenated head is part of a path only when it is written
without surrounding whitespace and followed by `::` — the rule of
[path-and-access.md](expressions/path-and-access.md) — so
`total - Math::PI` remains subtraction.

## Types — `NamedType`

```text
NamedType        = Path [ TypeArgs ] ;
```

A named type is a path, so `User`, `ui::hstack::HStack`, and
`lsp-server::config::Config` parse in every type position; type arguments
follow the path unchanged (`TypeArgs`).

## Expressions — paths

```text
PrimaryExpression = Literal | TemplateLiteral | Identifier | PathExpr
                  | "(" Expression ")" | TupleLiteral
                  | ListLiteral | MapLiteral
                  | IfExpression | MatchExpression | PanicExpression
                  | ClosureExpression | ComponentCall
                  | SpawnExpression | LockExpression ;
PathExpr         = Path [ "(" [ ArgumentList ] ")" ] ;
PanicExpression   = "panic" "(" Expression ")" ;
Place             = Identifier | Place "." Identifier | Place "." IntegerLiteral
                  | Place "[" Expression "]" | "(" Place ")" ;
```

`PathExpr` replaces `VariantPath`: the same production now writes enum
variants (`Theme::Dark`, `Shape::Circle(2)`), qualified module paths
(`form::validate(1, 2)`), package paths (`lsp-server::run()`), and built-in
or trait-associated names (`Math::max(a, b)`, `Area::area(r)`). The optional
argument list applies to the whole path exactly as it did to `VariantPath`.

## Nonterminals index

- The **Program and modules** row lists: `Program`, `HeaderItem`,
  `Declaration`, `Entry`, `EntryPoint`, `ModDecl`, `ModuleBody`, `UseDecl`,
  `PubUse`, `UseTarget`, `UsePath`, `UseGroup`, `UseGlob`,
  `UseEntry`, `PubUseLocal`, `PathHead`, `Path`, `PackageName`.
- The **Types** row: `NamedType` is defined as above (`Path [ TypeArgs ]`).
- The **Expressions** row: `VariantPath` is removed; `PathExpr` takes its
  place beside `PrimaryExpression`.
