# Grammar

This chapter gives the complete syntax of Xulo in EBNF. Under
[README §3](README.md) the grammar is **authoritative for syntax** and prose is
**authoritative for meaning**: the productions decide which token sequences are
well-formed, the chapters they are drawn from decide what those constructs
mean, and any disagreement MUST be reported rather than resolved silently.
Every nonterminal is defined exactly once, in the section that introduces it;
the [production index](#production-index) maps each one to its section.

## Notation

Grammar fragments use the EBNF of ISO/IEC 14977 and appear in `text` blocks.
Examples of Xulo source appear in `xulo` blocks and are non-normative.

| Form | Reading |
|------|---------|
| `name = expr ;` | definition of a nonterminal |
| `a \| b` | alternatives |
| `[ x ]` | `x` is optional |
| `{ x }` | `x` repeats zero or more times |
| `( x )` | grouping |
| `"text"` | a terminal, matched exactly |
| `? … ?` | a character class or lexical condition |
| `(* … *)` | a comment, not part of the grammar |

- **Naming.** Nonterminals are `CamelCase`; terminals are quoted source
  spellings, keywords included. Productions are unordered — a nonterminal MAY
  be referenced before it is defined — and each is defined exactly once.
- Juxtaposition denotes concatenation. Whitespace and comments separate tokens
  and carry no meaning; the complete lexical rules are in
  [lexical-structure.md](lexical-structure.md).

## Lexical structure

A source file is a sequence of tokens; whitespace and comments are discarded
before parsing, and every production below is written over that token stream.

```text
SourceFile     = { Skipped } [ TokenSequence ] ;
TokenSequence  = Token { [ Skipped ] Token } [ Skipped ] ;
Skipped        = Whitespace | Comment ;
Whitespace     = ? a space, a tab, or a line terminator (LF or CRLF) ? ;
Comment        = "//" ? characters up to the next line terminator ?
               | "/*" ? characters up to the first "*/", not nested ? ;
Token          = Identifier | Keyword | ReservedWord | Literal | Operator | Delimiter ;
```

### Identifiers

```text
Identifier = IdentStart { IdentPart } ;
IdentStart = Letter | "_" ;
IdentPart  = Letter | "_" | Digit ;
Letter     = ? one of the 52 ASCII letters, "A" to "Z" and "a" to "z" ? ;
Digit      = ? one of the 10 ASCII digits, "0" to "9" ? ;
HexDigit   = Digit | ? one of "a" to "f" or "A" to "F" ? ;
OctDigit   = ? one of "0" to "7" ? ;
```

Identifiers are ASCII, and `_` alone is the wildcard name: it lexes as an
identifier but MUST NOT be an ordinary name ([names.md](names.md)). A run is
formed first, then classified as keyword, reserved word, or identifier.

### Keywords

The language has exactly the following closed set of 58 keywords; every other
word is an ordinary identifier unless it is reserved.

```text
Keyword = "and" | "as" | "async" | "await" | "bool" | "break" | "const"
        | "continue" | "copy" | "defer" | "else" | "enum" | "f32" | "f64"
        | "false" | "float" | "fn" | "for" | "i8" | "i16" | "i32" | "i64"
        | "if" | "impl" | "in" | "int" | "is" | "let" | "lock" | "match"
        | "mod" | "move" | "mut" | "not" | "number" | "null" | "or" | "panic"
        | "pub" | "return" | "self" | "shared" | "spawn" | "string" | "struct"
        | "super" | "trait" | "true" | "type" | "u8" | "u16" | "u32" | "u64"
        | "unit" | "unknown" | "use" | "where" | "while" ;
```

A `ReservedWord` is a word whose spelling appears in the reserved list of
[lexical-structure.md](lexical-structure.md); it is neither a keyword nor an
identifier, and using one as a name is ill-formed. The reserved list excludes
every keyword above. `State`, `Store`, `Effect`, and `Environment` are
contextual: they are meaningful only directly after `@`.

### Literals

```text
Literal          = IntegerLiteral | FloatLiteral | StringLiteral
                 | TemplateLiteral | BooleanLiteral | NullLiteral ;
IntegerLiteral   = DecInt | HexInt | BinInt | OctInt ;
FloatLiteral     = DecInt "." DecInt ;
DecInt           = Digit { Digit } ;
HexInt           = "0" ( "x" | "X" ) HexDigit { HexDigit } ;
BinInt           = "0" ( "b" | "B" ) ( "0" | "1" ) { "0" | "1" } ;
OctInt           = "0" ( "o" | "O" ) OctDigit { OctDigit } ;
BooleanLiteral   = "true" | "false" ;
NullLiteral      = "null" ;

DQuote           = """" ;            (* the double-quote character *)
SQuote           = "'" ;             (* the single-quote character *)
Backslash        = "\" ;             (* the backslash character *)
Backtick         = "`" ;             (* the backtick character *)

StringLiteral    = DQuote { StringChar | EscapeSeq } DQuote
                 | SQuote { StringChar | EscapeSeq } SQuote ;
StringChar       = ? any character except a line terminator, a quote,
                        or a backslash ? ;
EscapeSeq        = Backslash ( DQuote | SQuote | Backslash | "n" | "t" | "r" )
                 | Backslash "u" HexDigit HexDigit HexDigit HexDigit
                 | Backslash "u" "{" HexDigit { HexDigit } "}" ;

TemplateLiteral  = Backtick { TemplateText | TemplateEscape | Interpolation }
                      Backtick ;
TemplateText     = ? any character except a backtick, a backslash, or the two
                        characters "${" ? ;
TemplateEscape   = EscapeSeq | Backslash Backtick | Backslash "$" ;
Interpolation    = "${" Expression "}" ;
```

There are no digit separators, no exponents, and no suffixes; a string literal
MUST NOT contain a raw line terminator and neither quote form interpolates.
Templates MAY span lines, and `${ Expression }` is tokenized as ordinary source
with braces counted ([lexical-structure.md](lexical-structure.md)).

### Operators and delimiters

```text
Operator = WordOperator | SymbolOperator ;
WordOperator = "and" | "or" | "is" | "not" ;
SymbolOperator =
      "." | "?" | "!" | "=" | "==" | "!=" | "<" | ">" | "<=" | ">="
    | "+" | "-" | "*" | "/" | "%" | "**" | "<<" | ">>" | "&" | "|" | "^" | "~"
    | "=>" | "..<" | "..." | "?." | "??" | "@" | "$" | "::" | ":=" | "#" ;
Delimiter = "(" | ")" | "{" | "}" | "[" | "]" | "," | ":" | ";" ;
```

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

## Declarations

Every alternative of `Declaration` MAY carry `pub`, except `impl` and
`TestDecl`: `impl` introduces no name and is never part of an interface, and
a `#Test` declaration is never part of an interface either
([visibility.md](modules/visibility.md)); the four component declarations
carry no visibility marker either
([state.md](components/state.md)). `async` follows `pub` and precedes `fn`.
The five item declarations — `FnDecl`, `StructDecl`, `EnumDecl`, `TraitDecl`,
and `ImplDecl` — MAY carry an attribute list, written outermost, before `pub`
and `async`; which attributes are defined, and on which of them, is a
checking rule and not a grammatical one
([traits.md](types/traits.md)). Generic parameters are declared once,
directly after the name, by functions,
`struct`s, `enum`s, type aliases, and `impl` blocks — never by a `trait`,
whose `GenericParams?` exists only so that an attempt can be diagnosed
(`E0810`)
([generics.md](types/generics.md), [traits.md](types/traits.md)).

```text
Attribute     = "#" Identifier [ "(" [ AttributeArgs ] ")" ] ;
AttributeArgs = AttributeArg { "," AttributeArg } [ "," ] ;
AttributeArg  = Identifier ;
AttributeList = { Attribute } ;

FnDecl = [ AttributeList ] [ "pub" ] [ "async" ] "fn" Identifier
         GenericParams? "(" [ ParameterList ] ")" [ ":" Type ]
         WhereClause? Block ;

ParameterList = Parameter { "," Parameter } [ "," ] ;
Parameter     = Receiver | Identifier ":" [ "mut" | "shared" ] Type
                              [ "=" Expression ] ;
Receiver      = "self" | "mut" "self" ;

StructDecl = [ AttributeList ] [ "pub" ] "struct" Identifier GenericParams?
             "{" [ FieldList ] "}" ;
FieldList  = Field { [ "," ] Field } [ "," ] ;
Field      = [ "pub" ] Identifier ":" Type ;

EnumDecl    = [ AttributeList ] [ "pub" ] "enum" Identifier GenericParams?
              "{" [ VariantList ] "}" ;
VariantList = Variant { [ "," ] Variant } [ "," ] ;
Variant     = Identifier [ VariantPayload ] ;
VariantPayload = "(" ( TypeList | NameTypeList ) ")" ;
NameTypeList   = NamedPayload { "," NamedPayload } ;
NamedPayload   = Identifier ":" Type ;

TraitDecl    = [ AttributeList ] [ "pub" ] "trait" Identifier GenericParams?
               "{" [ TraitMethods ] "}" ;
TraitMethods = TraitMethod { ";" TraitMethod } [ ";" ] ;
TraitMethod  = [ "async" ] "fn" Identifier "(" [ ParameterList ] ")" ":" Type ;

ImplDecl  = [ AttributeList ] "impl" GenericParams? [ Identifier "for" ]
            Type "{" { FnDecl } "}" ;
TypeAlias = [ "pub" ] "type" Identifier GenericParams? "=" Type ;
```

A parameter MUST be annotated; `mut T` and `shared T` are parameter modes, not
types, whether the parameter belongs to a declaration or to a function type
(`FnTypeParam`) ([function-types.md](types/function-types.md),
[concurrency.md](concurrency.md)). A `,` between struct fields and between
variants is optional and a trailing `,` is allowed; a variant's payload slots
are all positional or all named, never mixed ([enums.md](types/enums.md)).
Inside a `trait`, signatures are separated by `;` with the final `;` optional,
and every trait method declares a return type ([traits.md](types/traits.md));
an attribute list on a `trait` method is not grammatical — the list belongs
to declarations, and `TraitMethod` is a signature, not a declaration.

```text
LetDecl = [ "pub" ] "let" ( MutableBinding | SimpleBinding
                           | DestructuringBinding ) ;
MutableBinding       = "mut" Identifier [ ":" Type ] "=" [ "shared" ] Expression ;
SimpleBinding        = Identifier [ ":" Type ]
                       ( "=" [ "shared" ] Expression | ":=" Expression ) ;
DestructuringBinding = ( "{" IdentifierList "}"
                        | "(" IdentifierList ")" ) "=" Expression ;
IdentifierList       = Identifier { "," Identifier } [ "," ] ;

ConstDecl = [ "pub" ] "const" Identifier [ ":" Type ] "=" Expression ;

TestDecl = "#Test" [ "(" [ StringLiteral ] ")" ] [ "async" ] "fn"
           Identifier "(" ")" Block ;
```

`:=` appears only directly after `let`, never after `let mut` or `const`, and
`let x := e` means exactly `let mut x = e`
([let-and-assignment.md](statements/let-and-assignment.md)); initialization is
required. A `DestructuringBinding` deconstructs a `struct` with `{ … }` or a
tuple with `( … )`; the two forms take an initializer of the matching shape
and nothing else.

A `TestDecl` is a function declaration with a fixed shape: `#Test` precedes
its optional title group, `async`, and `fn` (never `async #Test fn`), the
parameter list and the return type are absent — the body is an implicit
`fn(): unit` — and it carries no `pub`, no generic parameters, and no
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
sequence `#Test` `("(" StringLiteral? ")")?` `async`? `fn` Identifier `(` `)`
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

### Component declarations

The four component declarations are valid only at the top level of a component
body — the body of a function whose declared return type is `View` — and their
forms are fixed by [state.md](components/state.md),
[effect.md](components/effect.md), and
[environment.md](components/environment.md).

```text
ComponentDecl   = StateDecl | StoreDecl | EffectDecl | EnvironmentDecl ;
StateDecl       = "@" "State" "let" Identifier [ ":" Type ] "=" Expression ;
StoreDecl       = "@" "Store" "let" "{" IdentifierList "}" "=" Expression ;
EffectDecl      = "@" "Effect" Expression [ "," Expression ] ;
EnvironmentDecl = "@" "Environment" "let" Identifier ":" Type ;
```

`@State` and `@Store` take an initializer; `@Environment` takes a REQUIRED
type annotation and none; `@Effect` takes a value plus an optional dependency
list after a `,`. The two `@Effect` operands are written as plain expressions
so that a wrong shape — `@Effect fn(): int { … }`, `@Effect fn() { … }, count`
— parses and is rejected by the checker (`E0709`, `E0708`) rather than
failing the parse. None of the four accepts `mut`, `pub`, or `:=`.

## Types

`?` binds tighter than `&`, which binds tighter than `|`, and parentheses group
([types/README.md](types/README.md)). The primitive value types — `bool`,
`string`, `number`, `int`, `float`, the fixed-bit numerics, `null`, `unit`,
`unknown` — are keywords written through `PrimitiveType`; every other type
name — `View`, the built-in composites, and every user-declared name — is an
ordinary identifier written through `NamedType`. The primitives are listed in
[primitive-types.md](types/primitive-types.md).

```text
Type             = UnionType ;
UnionType        = IntersectionType { "|" IntersectionType } ;
IntersectionType = PostfixType { "&" PostfixType } ;
PostfixType      = PrimaryType { "?" } ;
PrimaryType      = NamedType | PrimitiveType | "(" Type ")" | TupleType
                 | FunctionType | LiteralType ;
NamedType        = Path [ TypeArgs ] ;
PrimitiveType    = "bool" | "string" | "number" | "int" | "float"
                 | "i8" | "i16" | "i32" | "i64"
                 | "u8" | "u16" | "u32" | "u64"
                 | "f32" | "f64" | "null" | "unit" | "unknown" ;
LiteralType      = StringLiteral | IntegerLiteral | FloatLiteral
                 | BooleanLiteral ;

TypeArgs       = "<" TypeList ">" ;
TypeList       = Type { "," Type } ;
TupleType      = "(" Type "," Type { "," Type } [ "," ] ")" ;
FunctionType   = [ "async" ] "fn" "(" [ FnTypeParams ] ")" [ ":" Type ] ;
FnTypeParams   = FnTypeParam { "," FnTypeParam } ;
FnTypeParam    = [ Identifier ":" ] [ "mut" | "shared" ] Type ;

GenericParams  = "<" GenericParam { "," GenericParam } [ "," ] ">" ;
GenericParam   = Identifier [ ":" TraitBound ] ;
TraitBound     = Type { "+" Type } ;
WhereClause    = "where" WhereItem { "," WhereItem } ;
WhereItem      = Identifier ":" TraitBound ;
```

Because `>>` lexes as one token, two closing angle brackets MUST NOT be written
adjacently: a space is REQUIRED between them, as in `List<List<T> >`
([generics.md](types/generics.md)). Bounds are written inline `<T: Area>` or
in a `where` clause, joined with `+` when a parameter has several. A
`WhereClause` follows a function's signature only — every other parameterized
declaration uses the inline form — and a bound never carries type arguments
(`E0811`).

## Expressions

The precedence table of [expressions/README.md](expressions/README.md) is
mirrored below level by level, loosest binding first; the productions encode
each level exactly as that table orders it.

```text
Expression           = TernaryExpression ;
AssignmentExpression = TernaryExpression [ "=" AssignmentExpression ] ;
TernaryExpression    = LogicalOrExpression [ "?" Expression ":"
                                             TernaryExpression ] ;
LogicalOrExpression  = LogicalAndExpression { "or" LogicalAndExpression } ;
LogicalAndExpression = NullishExpression { "and" NullishExpression } ;
NullishExpression    = EqualityExpression { "??" EqualityExpression } ;
EqualityExpression   = IsExpression
                       { ( "==" | "!=" ) IsExpression } ;
IsExpression         = RelationalExpression
                       [ "is" [ "not" ] Type ] ;
RelationalExpression = RangeExpression
                        [ ( "<" | ">" | "<=" | ">=" ) RangeExpression ] ;
RangeExpression      = BitOrExpression [ ( "..<" | "..." ) BitOrExpression ] ;
BitOrExpression      = BitXorExpression { "|" BitXorExpression } ;
BitXorExpression     = BitAndExpression { "^" BitAndExpression } ;
BitAndExpression     = ShiftExpression { "&" ShiftExpression } ;
ShiftExpression      = AdditiveExpression
                       { ( "<<" | ">>" ) AdditiveExpression } ;
AdditiveExpression   = MultiplicativeExpression
                       { ( "+" | "-" ) MultiplicativeExpression } ;
MultiplicativeExpression = PowerExpression
                       { ( "*" | "/" | "%" ) PowerExpression } ;
PowerExpression      = UnaryExpression [ "**" PowerExpression ] ;
UnaryExpression      = PostfixExpression
                     | ( "!" | "-" | "~" | "await" | "move" | "copy" )
                       UnaryExpression ;
PostfixExpression    = PrimaryExpression { PostfixOperand } ;
PostfixOperand       = [ CallTypeArgs ] "(" [ ArgumentList ] ")"
                     | "[" Expression "]"
                     | "." ( Identifier | IntegerLiteral )
                     | "?." ( Identifier | IntegerLiteral )
                     | "?" ;
CallTypeArgs         = "<" [ TypeList ] ">" ;
```

Relational, range, and type-test operators are non-associative: `a < b < c`,
`0..<1..<2`, and `x is int is string` are not well-formed
([operators.md](expressions/operators.md)).
`move` and `copy` are prefix operators whose operand and position are
restricted by [memory-and-runtime.md](memory-and-runtime.md).

`?` follows an expression in two roles. When `Expression ":"` follows it —
whenever a complete ternary parses — it is the ternary operator of
`TernaryExpression`; otherwise it is the postfix propagation operator, whose
typing and failure rules are in [error-handling.md](error-handling.md). The
postfix reading survives only when the ternary reading fails, so
`c ? a : b` is never ambiguous with `f()?`.

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
nested-bracket rule already requires: `first<Map<string, int> >(m)` is
well-formed and `first<Map<string, int>>(m)` is not, the latter failing to
parse with a note to insert the space. Typing and diagnostics are in
[expressions/calls.md](expressions/calls.md).

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

ArgumentList = Argument { "," Argument } [ "," ] ;
Argument     = [ Identifier ":" ] ( "$" Place | Expression ) ;

ListLiteral   = "[" [ ListElement { "," ListElement } [ "," ] ] "]" ;
ListElement   = Spread | Expression ;
Spread        = "..." Expression ;
MapLiteral    = TypedMap | BraceMap ;
TypedMap      = "Map" TypeArgs "{" [ MapEntry { "," MapEntry } [ "," ] ] "}" ;
BraceMap      = "{" [ MapField { "," MapField } [ "," ] ] "}" ;
MapEntry      = Expression ":" Expression ;
MapField      = Spread | Identifier ":" Expression ;
TupleLiteral  = "(" Expression "," Expression { "," Expression } [ "," ] ")" ;

ClosureExpression = FunctionExpression | ArrowClosure ;
FunctionExpression = [ "async" ] "fn" "(" [ ClosureParamList ] ")"
                     [ ":" Type ] Block ;
ArrowClosure       = [ "async" ] ArrowParams [ ":" Type ] "=>"
                     ( Expression | Block ) ;
ArrowParams        = "(" [ ClosureParamList ] ")" | Identifier ;
ClosureParamList   = ClosureParam { "," ClosureParam } [ "," ] ;
ClosureParam       = Identifier [ ":" Type ] ;
```

`$` prefixes a place in an argument position and binds a state variable of the
enclosing component; it is an argument form and nothing else. The place is
written as a `Place` so that a dotted name — `$user.name` — parses and is
rejected by the checker rather than failing the parse, since only a single
state cell may be bound
([binding.md](components/binding.md)). Brace-map keys are identifiers only;
a map keyed by other expressions uses the typed form `Map<K, V>{ … }`.

The spread and the closed range share one token: an element of a list
literal or a brace map that *starts* with `...` is a spread, while elsewhere `...` is
the range operator, so `[...xs]` spreads `xs` but `[1...5]` is one range
element ([lexical-structure.md](lexical-structure.md)).

```xulo
let a = 1 + 2 * 3 ** 2         // ** binds tighter than *, right-associative
let ok = x >= 0 and x <= 10    // word operators only
let ys = [...xs, 4]            // spread inside a literal
```

## Patterns

A `match` arm binds with a pattern. A pattern MAY be a disjunction of
alternatives, and an arm MAY carry a guard; the first arm whose pattern
matches and whose guard, when present, evaluates to `true` wins, and
exhaustiveness is a checking rule
([control-flow.md](expressions/control-flow.md)).

```text
Pattern        = PatternAlt { "|" PatternAlt } ;
PatternAlt     = "_" | LiteralPattern | RangePattern | VariantPattern
               | StructPattern | TypePattern | Identifier
               | "(" Pattern ")" ;
LiteralPattern = IntegerLiteral | FloatLiteral | StringLiteral
               | BooleanLiteral | "null" ;
RangePattern   = ( IntegerLiteral | FloatLiteral ) ( "..<" | "..." )
                                 ( IntegerLiteral | FloatLiteral ) ;
VariantPattern = Identifier "::" Identifier [ "(" [ PatternList ] ")" ] ;
StructPattern  = Identifier "(" [ PatternList ] ")" ;
TypePattern    = PatternType ( "_" | Identifier ) ;
PatternType    = PatternDesignator [ "?" ] { "|" PatternDesignator [ "?" ] } ;
PatternDesignator = PrimitiveType | Identifier ;
PatternList    = Pattern { "," Pattern } [ "," ] ;
```

`_` is the wildcard and binds nothing; it lexes as an identifier, so the
wildcard and a binding of the same spelling are one alternative. Enum variants
use `::` in patterns exactly as in expressions, and the operands of a range
pattern are numeric literals. A type pattern is a type designator followed by
a binder — `string s`, `unknown _`, `string? p` — and the designator admits no
type-argument list
([control-flow.md](expressions/control-flow.md)).

`|` does double duty: it joins the designators of a union type pattern and
separates the alternatives of an or-pattern. The designator chain is read
first when a binder follows it — `string | int v` is one union type pattern,
as before — and `|` separates alternatives when no binder follows: `0 | 1`,
`A | B`, `Shape::Circle(_) | Shape::Dot(_)`. Parentheses group a single
alternative and settle the remaining case: `(A) | B c` is two alternatives,
while `A | B c` remains one union type pattern.

## Statements

A statement is a construct that produces no meaningful value. The grammar is
newline-insensitive, so `;` is an optional terminator: a statement ends when
the next token cannot continue it.

```text
Statement = ( FnDecl | LetDecl | ConstDecl | ComponentDecl
            | ReturnStmt | BreakStmt | ContinueStmt | DeferStmt
            | Assignment | ExpressionStatement
            | ForStmt | WhileStmt ) StatementEnd ;
StatementEnd      = [ ";" ] ;
ReturnStmt        = "return" [ Expression ] ;
BreakStmt         = "break" ;
ContinueStmt      = "continue" ;
DeferStmt         = "defer" ( Block | Expression ) ;
Assignment        = Place "=" AssignmentExpression ;
ExpressionStatement = TernaryExpression ;
Block             = "{" { Statement } "}" ;
```

An assignment appears only through `Assignment`, so `ExpressionStatement` is
every level below assignment, and its target MUST be a place
([expressions/README.md](expressions/README.md)). `Expression` itself stops at
`TernaryExpression`, so no production other than `Assignment` introduces `=`;
`Assignment`'s right-hand side is `AssignmentExpression`, which recurses so that
`a = b = c` groups as `a = (b = c)`. A block's value is its final
expression, kept only if not followed by `;`
([return-and-block.md](statements/return-and-block.md)).

`defer e` registers its operand — an expression or a block — for evaluation
when control leaves the enclosing block, and discards the value it produces
([return-and-block.md](statements/return-and-block.md)). An operand beginning
with `{` is read as a block, not a map; write a map literal in parentheses
when one must be deferred.

## Control flow

`if` and `match` are expressions; `for`, `while`, `break`/`continue`, and
`defer` are statements; `panic` is an expression, so in statement position it is
an `ExpressionStatement` whose value is discarded
([statements/expression-statements.md](statements/expression-statements.md)).

```text
IfExpression  = "if" Expression Block [ "else" ( Block | IfExpression ) ] ;
MatchExpression = "match" Expression "{" [ MatchArmList ] "}" ;
MatchArmList  = MatchArm { [ "," ] MatchArm } [ "," ] ;
MatchArm      = Pattern [ Guard ] "=>" ( Expression | Block ) ;
Guard         = "if" Expression ;

ForStmt   = "for" Identifier "in" Expression Block ;
WhileStmt = "while" Expression Block ;
```

`else` may be followed directly by another `if`, which chains without limit.
Arms are separated by newlines or optional `,`; each body is an expression or
a block, and each arm MAY carry a guard — `if condition` between its pattern
and `=>` — which is evaluated only when the pattern has matched. The `=>` that
ends an arm is the first one written outside any parentheses, brackets, or
braces, so a closure cannot stand unparenthesized in a guard: write
`x if ((n) => n > 0) => …`, and a `match` or `if` inside a guard is unaffected
because its own `=>` and braces are nested. Failure propagation with `?` and
the stopping `panic` expression are specified in
[error-handling.md](error-handling.md).

## Components and UI

A component invocation is an identifier followed by an argument list, a block
of children, or both; at least one of the two MUST be present. The children
block holds child items, which are not statements
([view-syntax.md](components/view-syntax.md)). `ComponentCall` recognizes the
forms that carry a block, since a trailing block attaches to the invocation and
cannot be written any other way; an invocation written with an argument list
and no block — `Text("Hi")`, `first<int>([1, 2, 3])` — is parsed as an
ordinary call through `PostfixOperand`, and the two read the same.

```text
ComponentCall  = Identifier [ CallTypeArgs ] "(" [ ArgumentList ] ")" ComponentBlock
               | Identifier ComponentBlock ;
ComponentBlock = "{" { ChildItem } "}" ;
ChildItem      = Expression | ForStmt ;
```

A component invocation, an `if`, and a string, `View`, or `List<View>`
expression reach `ChildItem` through `Expression`; `for` is written through
`ForStmt` because a loop is a statement. Anything else in a block — a binding,
an assignment, a `return`, an expression of another type — is a compile-time
error, and those restrictions are typing rules stated in
[view-syntax.md](components/view-syntax.md).

```xulo
fn Counter(): View {
  @State let count: int = 0
  VStack(spacing: 8) {
    Text(str(count))
    Button("Inc", onClick: fn() { count = count + 1 })
  }
}
```

## Concurrency

`spawn` has five forms: four run a block on a chosen dispatch mode, the fifth
starts a service. `lock` guards a shared binding, which `shared` marks only
in binding and parameter forms ([concurrency.md](concurrency.md)).

```text
SpawnExpression = "spawn" SpawnTarget "async" Block
                | "spawn" "." "process" Type "{" [ FieldInitList ] "}" ;
SpawnTarget     = [ "." ( "thread" | "process" | "on" "(" Expression ")" ) ] ;
LockExpression  = "lock" Identifier Block ;
FieldInitList   = FieldInit { "," FieldInit } [ "," ] ;
FieldInit       = Identifier ":" Expression ;
```

A shared binding is written in a `let` initializer (`let state = shared Expr`)
and a shared parameter as `name: shared T`; the operand of `lock` MUST be a
shared binding.

```xulo
async fn main() {
  let job = spawn.thread async { compute() }
  print(await job)
  lock state { state.total = state.total + 1 }
}
```

## Production index

Every nonterminal of this chapter and the section that defines it; each
production itself appears exactly once, in its own section.

| Nonterminal | Section |
|-------------|---------|
| `SourceFile`, `TokenSequence`, `Skipped`, `Whitespace`, `Comment`, `Token`, `Identifier`, `IdentStart`, `IdentPart`, `Letter`, `Digit`, `HexDigit`, `OctDigit`, `Keyword`, `Literal`, `IntegerLiteral`, `FloatLiteral`, `DecInt`, `HexInt`, `BinInt`, `OctInt`, `BooleanLiteral`, `NullLiteral` | [Lexical structure](#lexical-structure) |
| `StringLiteral`, `StringChar`, `EscapeSeq`, `TemplateLiteral`, `TemplateText`, `TemplateEscape`, `Interpolation` | [Lexical structure](#lexical-structure) |
| `DQuote`, `SQuote`, `Backslash`, `Backtick`, `Operator`, `WordOperator`, `SymbolOperator`, `Delimiter` | [Lexical structure](#lexical-structure) |
| `Program`, `HeaderItem`, `Declaration`, `Entry`, `EntryPoint`, `ModDecl`, `ModuleBody`, `UseDecl`, `PubUse`, `UseTarget`, `UsePath`, `UseGroup`, `UseGlob`, `UseEntry`, `PubUseLocal`, `PathHead`, `Path`, `PackageName` | [Program and modules](#program-and-modules) |
| `Attribute`, `AttributeArgs`, `AttributeArg`, `AttributeList`, `FnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl`, `TestDecl` | [Declarations](#declarations) |
| `ComponentDecl`, `StateDecl`, `StoreDecl`, `EffectDecl`, `EnvironmentDecl` | [Declarations](#declarations) |
| `Type`, `UnionType`, `IntersectionType`, `PostfixType`, `PrimaryType`, `NamedType`, `PrimitiveType`, `LiteralType`, `TypeArgs`, `TypeList`, `TupleType`, `FunctionType`, `FnTypeParams`, `FnTypeParam`, `GenericParams`, `GenericParam`, `TraitBound`, `WhereClause`, `WhereItem` | [Types](#types) |
| `Expression`, `AssignmentExpression`, `TernaryExpression`, `LogicalOrExpression`, `LogicalAndExpression`, `NullishExpression`, `EqualityExpression`, `IsExpression`, `RelationalExpression`, `RangeExpression`, `BitOrExpression`, `BitXorExpression`, `BitAndExpression`, `ShiftExpression`, `AdditiveExpression`, `MultiplicativeExpression`, `PowerExpression`, `UnaryExpression`, `PostfixExpression`, `PostfixOperand`, `CallTypeArgs` | [Expressions](#expressions) |
| `PrimaryExpression`, `PathExpr`, `PanicExpression`, `Place`, `ArgumentList`, `Argument`, `ListLiteral`, `ListElement`, `Spread`, `MapLiteral`, `TypedMap`, `BraceMap`, `MapEntry`, `MapField`, `TupleLiteral`, `ClosureExpression`, `FunctionExpression`, `ArrowClosure`, `ArrowParams`, `ClosureParamList`, `ClosureParam` | [Expressions](#expressions) |
| `Pattern`, `PatternAlt`, `LiteralPattern`, `RangePattern`, `VariantPattern`, `StructPattern`, `TypePattern`, `PatternType`, `PatternDesignator`, `PatternList` | [Patterns](#patterns) |
| `Statement`, `StatementEnd`, `ReturnStmt`, `BreakStmt`, `ContinueStmt`, `DeferStmt`, `Assignment`, `ExpressionStatement`, `Block` | [Statements](#statements) |
| `IfExpression`, `MatchExpression`, `MatchArmList`, `MatchArm`, `Guard`, `ForStmt`, `WhileStmt` | [Control flow](#control-flow) |
| `ComponentCall`, `ComponentBlock`, `ChildItem` | [Components and UI](#components-and-ui) |
| `SpawnExpression`, `SpawnTarget`, `LockExpression`, `FieldInitList`, `FieldInit` | [Concurrency](#concurrency) |

## Consistency rules

- The grammar is newline-insensitive, `;` is an optional statement terminator,
  and lexing is longest match ([lexical-structure.md](lexical-structure.md)).
- A tuple type or literal needs at least two comma-separated elements —
  `( T )` and `( e )` only group, `f( a, b )` passes two arguments, and `()`
  does not parse — and there is no `export` keyword: module-level and member
  visibility are both written with `pub`, the default being private.
- `{` begins a block, a brace map, or a component's children block, told
  apart by their contents, and an expression statement MUST NOT begin with `{`
  ([statements/README.md](statements/README.md)).
- When `?` follows an expression it is the ternary operator if
  `Expression ":"` follows it, and the postfix propagation operator otherwise:
  `c ? a : b` parses as written, `f()?` propagates
  ([error-handling.md](error-handling.md)).
