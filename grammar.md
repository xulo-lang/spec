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

The language has exactly the following closed set of 38 keywords; every other
word is an ordinary identifier unless it is reserved.

```text
Keyword = "and" | "as" | "async" | "await" | "break" | "const"
        | "continue" | "copy" | "else" | "enum" | "false" | "fn"
        | "for" | "from" | "if" | "impl" | "in" | "import" | "let" | "lock"
        | "match" | "move" | "mut" | "null" | "or" | "panic" | "pub"
        | "return" | "self" | "shared" | "spawn" | "struct" | "trait" | "true"
        | "type" | "use" | "where" | "while" ;
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
WordOperator = "and" | "or" ;
SymbolOperator =
      "." | "?" | "!" | "=" | "==" | "!=" | "<" | ">" | "<=" | ">="
    | "+" | "-" | "*" | "/" | "%" | "**" | "<<" | ">>" | "&" | "|" | "^" | "~"
    | "=>" | "..<" | "..." | "?." | "??" | "@" | "$" | "::" | ":=" ;
Delimiter = "(" | ")" | "{" | "}" | "[" | "]" | "," | ":" | ";" ;
```

This is the complete inventory. `::` is the enum-variant path separator and
`:=` is the mutable-binding sugar of a `let` declaration; the connectives are
the word operators `and`, `or` and the prefix `!`, and no other language's
symbolic spellings for them exist. At each position the lexer takes the longest
token in the inventory.

## Program and modules

A source file is a module, and `Program` is the whole file. Imports MUST
appear in the file header; `pub use` MAY appear anywhere among the
declarations ([source-files.md](modules/source-files.md)). Visibility is
written with `pub`, and the default is private everywhere.

```text
Program     = { ImportDecl } { Declaration } [ Entry ] ;
Declaration = FnDecl | StructDecl | EnumDecl | TraitDecl | ImplDecl
            | TypeAlias | LetDecl | ConstDecl | PubUse ;

ImportDecl      = "import" ( TypeImport | NamedImport
                           | NamespaceImport | SideEffectImport ) ;
TypeImport      = "type" NamedImports "from" StringLiteral ;
NamedImport     = NamedImports "from" StringLiteral ;
NamespaceImport = "*" "as" Identifier "from" StringLiteral ;
SideEffectImport = StringLiteral ;
NamedImports    = "{" [ ImportEntry { "," ImportEntry } [ "," ] ] "}" ;
ImportEntry     = Identifier [ "as" Identifier ] ;

PubUse = "pub" "use" "{" [ Identifier { "," Identifier } [ "," ] ] "}"
         [ "from" StringLiteral ] ;
```

`Entry` is the entry point of an executable program — a module-level `main`
with the signature [source-files.md](modules/source-files.md) allows (`fn
main()`, `fn main(): View`, or `async fn main()`, no parameters). Since
`FnDecl` derives `main` too, the grammar constrains neither its form nor its
position; `Entry` names it in the top-level shape of `Program`.

```text
Entry      = EntryPoint ;
EntryPoint = [ "pub" ] [ "async" ] "fn" "main" "(" ")" [ ":" Type ] Block ;
```

## Declarations

Every declaration MAY carry `pub`; `async` follows `pub` and precedes `fn`.
Generic parameters are declared once, directly after the name
([generics.md](types/generics.md)).

```text
FnDecl = [ "pub" ] [ "async" ] "fn" Identifier GenericParams?
         "(" [ ParameterList ] ")" [ ":" Type ] WhereClause? Block ;

ParameterList = Parameter { "," Parameter } [ "," ] ;
Parameter     = Receiver | Identifier ":" [ "mut" | "shared" ] Type
                              [ "=" Expression ] ;
Receiver      = "self" | "mut" "self" ;

StructDecl = [ "pub" ] "struct" Identifier GenericParams?
             "{" [ FieldList ] "}" ;
FieldList  = Field { [ "," ] Field } [ "," ] ;
Field      = [ "pub" ] Identifier ":" Type ;

EnumDecl    = [ "pub" ] "enum" Identifier GenericParams?
              "{" [ VariantList ] "}" ;
VariantList = Variant { [ "," ] Variant } [ "," ] ;
Variant     = Identifier [ VariantPayload ] ;
VariantPayload = "(" ( TypeList | NameTypeList ) ")" ;
NameTypeList   = NamedPayload { "," NamedPayload } ;
NamedPayload   = Identifier ":" Type ;

TraitDecl    = [ "pub" ] "trait" Identifier GenericParams?
               "{" [ TraitMethods ] "}" ;
TraitMethods = TraitMethod { ";" TraitMethod } [ ";" ] ;
TraitMethod  = [ "async" ] "fn" Identifier "(" [ ParameterList ] ")" ":" Type ;

ImplDecl  = "impl" GenericParams? [ Identifier "for" ] Type "{" { FnDecl } "}" ;
TypeAlias = [ "pub" ] "type" Identifier GenericParams? "=" Type ;
```

A parameter MUST be annotated; `mut T` and `shared T` are parameter modes, not
types ([function-types.md](types/function-types.md),
[concurrency.md](concurrency.md)). A `,` between struct fields and between
variants is optional and a trailing `,` is allowed; a variant's payload slots
are all positional or all named, never mixed ([enums.md](types/enums.md)).
Inside a `trait`, signatures are separated by `;` with the final `;` optional,
and every trait method declares a return type ([traits.md](types/traits.md)).

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
```

`:=` appears only directly after `let`, never after `let mut` or `const`, and
`let x := e` means exactly `let mut x = e`
([let-and-assignment.md](statements/let-and-assignment.md)); initialization is
required. A `DestructuringBinding` deconstructs an object with `{ … }` or a
tuple with `( … )`; the two forms take an initializer of the matching shape
and nothing else.

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
EffectDecl      = "@" "Effect" ClosureExpression [ "," ListLiteral ] ;
EnvironmentDecl = "@" "Environment" "let" Identifier ":" Type ;
```

`@State` and `@Store` take an initializer; `@Environment` takes a REQUIRED
type annotation and none; `@Effect` takes a closure plus an optional
dependency list (a list literal after a `,`). None accepts `mut`, `pub`, or
`:=`.

## Types

`?` binds tighter than `&`, which binds tighter than `|`, and parentheses group
([types/README.md](types/README.md)). Primitive type names — `boolean`,
`string`, `number`, `int`, `float`, the fixed-bit numerics, `null`, `unit`,
`object`, `View` — are ordinary identifiers written through `NamedType`, listed
in [primitive-types.md](types/primitive-types.md).

```text
Type             = UnionType ;
UnionType        = IntersectionType { "|" IntersectionType } ;
IntersectionType = PostfixType { "&" PostfixType } ;
PostfixType      = PrimaryType { "?" } ;
PrimaryType      = NamedType | "(" Type ")" | TupleType | ObjectType
                 | FunctionType | LiteralType ;
NamedType        = Identifier [ TypeArgs ] ;
LiteralType      = StringLiteral | IntegerLiteral | FloatLiteral
                 | BooleanLiteral ;

TypeArgs       = "<" TypeList ">" ;
TypeList       = Type { "," Type } ;
TupleType      = "(" Type "," Type { "," Type } [ "," ] ")" ;
ObjectType     = "{" [ TypeField { "," TypeField } [ "," ] ] "}" ;
TypeField      = Identifier ":" Type ;
FunctionType   = [ "async" ] "fn" "(" [ FnTypeParams ] ")" [ ":" Type ] ;
FnTypeParams   = FnTypeParam { "," FnTypeParam } ;
FnTypeParam    = [ Identifier ":" ] Type ;

GenericParams  = "<" GenericParam { "," GenericParam } [ "," ] ">" ;
GenericParam   = Identifier [ ":" TraitBound ] ;
TraitBound     = Type { "+" Type } ;
WhereClause    = "where" WhereItem { "," WhereItem } ;
WhereItem      = Identifier ":" TraitBound ;
```

Because `>>` lexes as one token, two closing angle brackets MUST NOT be written
adjacently: a space is REQUIRED between them, as in `list<list<T> >`
([generics.md](types/generics.md)). Bounds are written inline `<T: Area>` or
in a `where` clause, joined with `+` when a parameter has several.

## Expressions

The precedence table of [expressions/README.md](expressions/README.md) is
mirrored below level by level, loosest binding first; the productions encode
each level exactly as that table orders it.

```text
Expression           = AssignmentExpression ;
AssignmentExpression = TernaryExpression [ "=" AssignmentExpression ] ;
TernaryExpression    = LogicalOrExpression [ "?" Expression ":"
                                             TernaryExpression ] ;
LogicalOrExpression  = LogicalAndExpression { "or" LogicalAndExpression } ;
LogicalAndExpression = NullishExpression { "and" NullishExpression } ;
NullishExpression    = EqualityExpression { "??" EqualityExpression } ;
EqualityExpression   = RelationalExpression
                       { ( "==" | "!=" ) RelationalExpression } ;
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
PostfixOperand       = "(" [ ArgumentList ] ")"
                     | "[" Expression "]"
                     | "." ( Identifier | IntegerLiteral )
                     | "?." ( Identifier | IntegerLiteral )
                     | "?" ;
```

Relational and range operators are non-associative: `a < b < c` and
`0..<1..<2` are not well-formed ([operators.md](expressions/operators.md)).
`move` and `copy` are prefix operators whose operand and position are
restricted by [memory-and-runtime.md](memory-and-runtime.md).

`?` follows an expression in two roles. When `Expression ":"` follows it —
whenever a complete ternary parses — it is the ternary operator of
`TernaryExpression`; otherwise it is the postfix propagation operator, whose
typing and failure rules are in [error-handling.md](error-handling.md). The
postfix reading survives only when the ternary reading fails, so
`c ? a : b` is never ambiguous with `f()?`.

```text
PrimaryExpression = Literal | TemplateLiteral | Identifier | VariantPath
                  | "(" Expression ")" | TupleLiteral
                  | ListLiteral | ObjectLiteral | MapLiteral
                  | IfExpression | MatchExpression | PanicExpression
                  | ClosureExpression | ComponentCall
                  | SpawnExpression | LockExpression ;
VariantPath       = Identifier "::" Identifier [ "(" [ ArgumentList ] ")" ] ;
PanicExpression   = "panic" "(" Expression ")" ;
Place             = Identifier | Place "." Identifier | Place "." IntegerLiteral
                  | Place "[" Expression "]" | "(" Place ")" ;

ArgumentList = Argument { "," Argument } [ "," ] ;
Argument     = [ Identifier ":" ] ( "$" Identifier | Expression ) ;

ListLiteral   = "[" [ ListElement { "," ListElement } [ "," ] ] "]" ;
ListElement   = Spread | Expression ;
Spread        = "..." Expression ;
ObjectLiteral = "{" [ ObjectField { "," ObjectField } [ "," ] ] "}" ;
ObjectField   = Spread | Identifier ":" Expression ;
MapLiteral    = "map" TypeArgs "{" [ MapEntry { "," MapEntry } [ "," ] ] "}" ;
MapEntry      = Expression ":" Expression ;
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

`$` prefixes a name in an argument position and binds a state variable of the
enclosing component; it is an argument form and nothing else
([binding.md](components/binding.md)). Object literal keys are identifiers
only; a value with arbitrary string keys uses `MapLiteral`.

The spread and the closed range share one token: an element of a list or
object literal that *starts* with `...` is a spread, while elsewhere `...` is
the range operator, so `[...xs]` spreads `xs` but `[1...5]` is one range
element ([lexical-structure.md](lexical-structure.md)).

```xulo
let a = 1 + 2 * 3 ** 2         // ** binds tighter than *, right-associative
let ok = x >= 0 and x <= 10    // word operators only
let ys = [...xs, 4]            // spread inside a literal
```

## Patterns

A `match` arm binds with a pattern. There are no OR-patterns and no guards:
the first arm whose pattern matches wins, and exhaustiveness is a checking rule
([control-flow.md](expressions/control-flow.md)).

```text
Pattern        = "_" | LiteralPattern | RangePattern | VariantPattern
               | StructPattern | Identifier ;
LiteralPattern = IntegerLiteral | FloatLiteral | StringLiteral
               | BooleanLiteral | "null" ;
RangePattern   = ( IntegerLiteral | FloatLiteral ) ( "..<" | "..." )
                                 ( IntegerLiteral | FloatLiteral ) ;
VariantPattern = Identifier "::" Identifier [ "(" [ PatternList ] ")" ] ;
StructPattern  = Identifier "(" [ PatternList ] ")" ;
PatternList    = Pattern { "," Pattern } [ "," ] ;
```

`_` is the wildcard and binds nothing; it lexes as an identifier, so the
wildcard and a binding of the same spelling are one alternative. Enum variants
use `::` in patterns exactly as in expressions, and the operands of a range
pattern are numeric literals.

## Statements

A statement is a construct that produces no meaningful value. The grammar is
newline-insensitive, so `;` is an optional terminator: a statement ends when
the next token cannot continue it.

```text
Statement = ( FnDecl | LetDecl | ConstDecl | ComponentDecl
            | ReturnStmt | BreakStmt | ContinueStmt
            | Assignment | ExpressionStatement
            | ForStmt | WhileStmt ) StatementEnd ;
StatementEnd      = [ ";" ] ;
ReturnStmt        = "return" [ Expression ] ;
BreakStmt         = "break" ;
ContinueStmt      = "continue" ;
Assignment        = Place "=" Expression ;
ExpressionStatement = TernaryExpression ;
Block             = "{" { Statement } "}" ;
```

An assignment appears only through `Assignment`, so `ExpressionStatement` is
every level below assignment, and its target MUST be a place
([expressions/README.md](expressions/README.md)). A block's value is its final
expression, kept only if not followed by `;`
([return-and-block.md](statements/return-and-block.md)).

## Control flow

`if` and `match` are expressions; `for`, `while`, and `break`/`continue` are
statements; `panic` is an expression, so in statement position it is an
`ExpressionStatement` whose value is discarded
([statements/expression-statements.md](statements/expression-statements.md)).

```text
IfExpression  = "if" Expression Block [ "else" ( Block | IfExpression ) ] ;
MatchExpression = "match" Expression "{" [ MatchArmList ] "}" ;
MatchArmList  = MatchArm { [ "," ] MatchArm } [ "," ] ;
MatchArm      = Pattern "=>" ( Expression | Block ) ;

ForStmt   = "for" Identifier "in" Expression Block ;
WhileStmt = "while" Expression Block ;
```

`else` may be followed directly by another `if`, which chains without limit.
Arms are separated by newlines or optional `,`; each body is an expression or
a block. Failure propagation with `?` and the stopping `panic` expression are
specified in [error-handling.md](error-handling.md).

## Components and UI

A component invocation is an identifier followed by an argument list, a block
of children, or both; at least one of the two MUST be present. The children
block holds child items, which are not statements
([view-syntax.md](components/view-syntax.md)).

```text
ComponentCall  = Identifier "(" [ ArgumentList ] ")" ComponentBlock
               | Identifier ComponentBlock ;
ComponentBlock = "{" { ChildItem } "}" ;
ChildItem      = Expression | ForStmt ;
```

A component invocation, an `if`, and a string, `View`, or `list<View>`
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
| `Program`, `Declaration`, `Entry`, `EntryPoint`, `ImportDecl`, `TypeImport`, `NamedImport`, `NamespaceImport`, `SideEffectImport`, `NamedImports`, `ImportEntry`, `PubUse` | [Program and modules](#program-and-modules) |
| `FnDecl`, `ParameterList`, `Parameter`, `Receiver`, `StructDecl`, `FieldList`, `Field`, `EnumDecl`, `VariantList`, `Variant`, `VariantPayload`, `NameTypeList`, `NamedPayload`, `TraitDecl`, `TraitMethods`, `TraitMethod`, `ImplDecl`, `TypeAlias`, `LetDecl`, `MutableBinding`, `SimpleBinding`, `DestructuringBinding`, `IdentifierList`, `ConstDecl` | [Declarations](#declarations) |
| `ComponentDecl`, `StateDecl`, `StoreDecl`, `EffectDecl`, `EnvironmentDecl` | [Declarations](#declarations) |
| `Type`, `UnionType`, `IntersectionType`, `PostfixType`, `PrimaryType`, `NamedType`, `LiteralType`, `TypeArgs`, `TypeList`, `TupleType`, `ObjectType`, `TypeField`, `FunctionType`, `FnTypeParams`, `FnTypeParam`, `GenericParams`, `GenericParam`, `TraitBound`, `WhereClause`, `WhereItem` | [Types](#types) |
| `Expression`, `AssignmentExpression`, `TernaryExpression`, `LogicalOrExpression`, `LogicalAndExpression`, `NullishExpression`, `EqualityExpression`, `RelationalExpression`, `RangeExpression`, `BitOrExpression`, `BitXorExpression`, `BitAndExpression`, `ShiftExpression`, `AdditiveExpression`, `MultiplicativeExpression`, `PowerExpression`, `UnaryExpression`, `PostfixExpression`, `PostfixOperand` | [Expressions](#expressions) |
| `PrimaryExpression`, `VariantPath`, `PanicExpression`, `Place`, `ArgumentList`, `Argument`, `ListLiteral`, `ListElement`, `Spread`, `ObjectLiteral`, `ObjectField`, `MapLiteral`, `MapEntry`, `TupleLiteral`, `ClosureExpression`, `FunctionExpression`, `ArrowClosure`, `ArrowParams`, `ClosureParamList`, `ClosureParam` | [Expressions](#expressions) |
| `Pattern`, `LiteralPattern`, `RangePattern`, `VariantPattern`, `StructPattern`, `PatternList` | [Patterns](#patterns) |
| `Statement`, `StatementEnd`, `ReturnStmt`, `BreakStmt`, `ContinueStmt`, `Assignment`, `ExpressionStatement`, `Block` | [Statements](#statements) |
| `IfExpression`, `MatchExpression`, `MatchArmList`, `MatchArm`, `ForStmt`, `WhileStmt` | [Control flow](#control-flow) |
| `ComponentCall`, `ComponentBlock`, `ChildItem` | [Components and UI](#components-and-ui) |
| `SpawnExpression`, `SpawnTarget`, `LockExpression`, `FieldInitList`, `FieldInit` | [Concurrency](#concurrency) |

## Consistency rules

- The grammar is newline-insensitive, `;` is an optional statement terminator,
  and lexing is longest match ([lexical-structure.md](lexical-structure.md)).
- A tuple type or literal needs at least two comma-separated elements —
  `( T )` and `( e )` only group, `f( a, b )` passes two arguments, and `()`
  does not parse — and there is no `export` keyword: module-level and member
  visibility are both written with `pub`, the default being private.
- `{` begins a block, an object literal, or a component's children block, told
  apart by their contents, and an expression statement MUST NOT begin with `{`
  ([statements/README.md](statements/README.md)).
- When `?` follows an expression it is the ternary operator if
  `Expression ":"` follows it, and the postfix propagation operator otherwise:
  `c ? a : b` parses as written, `f()?` propagates
  ([error-handling.md](error-handling.md)).
