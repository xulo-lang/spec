# Lexical Structure

This chapter defines how the characters of a `.xulo` file become the tokens
the parser consumes: encoding, whitespace and comments, token spellings, and
the disambiguation of competing spellings. Lexical structure decides what the
symbols are, never what they mean.

## Source Encoding

- Source files MUST be encoded in UTF-8, as also required by
  [README §3](README.md), and a multi-byte UTF-8 sequence never splits a token.
- A byte-order mark (U+FEFF) at the very start of a file is ignored; a BOM in
  any other position is an error where a token is expected.
- Line terminators are LF (`\n`) and CRLF (`\r\n`), and the two are equivalent:
  wherever this specification refers to a line — the reach of a line comment,
  for instance — either form counts as one line terminator.

## Whitespace and Statement Termination

Whitespace (space, tab, line terminator) separates tokens and is otherwise
insignificant; indentation is not significant — blocks are delimited by `{ }`.

The grammar is newline-insensitive: a newline is whitespace like any other,
and `;` is an optional statement terminator. A statement ends when the next
token cannot continue it. This is not automatic semicolon insertion — no token
is inserted, removed, or modified because of a line break, so reflowing a file
never changes how it parses.

## Comments

- A line comment is `//` up to the next line terminator, and is discarded
  before parsing. A block comment is `/* ... */`; it does not nest — the first
  `*/` ends the comment, and any `/*` inside it is ordinary comment text.
- An unterminated block comment is a lexical error, and there is no
  documentation-comment syntax: a comment carries no meaning to the language.

## Identifiers

```text
Identifier ::= Letter { Letter | Digit }
Letter     ::= 'A' .. 'Z' | 'a' .. 'z' | '_'
Digit      ::= '0' .. '9'
```

- Identifiers are ASCII: no accented letters, no other scripts, and no
  combining characters may appear in one.
- An identifier run is formed first (maximal munch: `iffy` is one identifier,
  not `if` followed by `fy`), then classified: a keyword spelling becomes that
  keyword, a reserved spelling a reserved word, anything else an identifier.
- `_` alone is the wildcard/discard name; it lexes as an identifier token but
  MUST NOT be used as an ordinary identifier. Its uses, and the casing
  conventions, are in [names.md](names.md).

## Keywords

The language has exactly the following closed set of 38 keywords; every other
word is an ordinary identifier unless it is reserved below. No keyword MAY be
used as one.

| Category | Keyword | Purpose |
|----------|---------|---------|
| Declarations | `const` | Declares a compile-time constant |
| | `enum` | Declares an enumeration type |
| | `fn` | Declares a function or a function-typed value |
| | `impl` | Declares the implementation of a trait for a type |
| | `let` | Declares a binding; immutable unless preceded by `mut` |
| | `mut` | Marks a binding or a borrow as mutable |
| | `self` | The receiver of a method inside an `impl` block |
| | `struct` | Declares a structure type |
| | `trait` | Declares a trait |
| | `type` | Declares a type alias |
| | `where` | Introduces a generic constraint clause after a signature |
| Control flow | `break` | Leaves the innermost enclosing `for` or `while` |
| | `continue` | Starts the next iteration of the innermost loop |
| | `else` | Introduces the alternative of an `if` |
| | `for` | Declares a counting or iterating loop |
| | `if` | Conditional expression or statement |
| | `in` | Separates the loop variable from the iterable in `for` |
| | `match` | Pattern matching over a scrutinee |
| | `panic` | Stops the program with an unrecoverable error |
| | `return` | Returns a value from the enclosing function |
| | `while` | Declares a condition-controlled loop |
| Modules | `as` | Renames an imported or re-exported name |
| | `from` | Names the module in an `import` declaration |
| | `import` | Brings names from another module into scope |
| | `pub` | Marks a declaration or member as publicly visible |
| | `use` | Re-exports a name from this or another module |
| Async and concurrency | `async` | Declares an asynchronous function or closure |
| | `await` | Unwraps a `Task<T>` inside an `async` body |
| | `lock` | Enters a critical section over `shared` state |
| | `shared` | Declares state that may be shared between tasks |
| | `spawn` | Starts a task, thread, process, or pooled job |
| Literals and operators | `and` | Logical conjunction (word operator) |
| | `false` | The `boolean` literal for falsity |
| | `null` | The null literal; an optional with no content |
| | `or` | Logical disjunction (word operator) |
| | `true` | The `boolean` literal for truth |
| Ownership | `copy` | Evaluates an expression and deep-copies the result |
| | `move` | Transfers ownership of a value out of its binding |

## Contextual Keywords

Four words are meaningful only directly after `@`, where they form the state
declarations of a component: `State`, `Store`, `Effect`, `Environment`, written
`@State`, `@Store`, `@Effect`, `@Environment` and valid only at the top level
of a component body (see the [components chapter](components/README.md)).
Everywhere else they are ordinary identifiers: in `let Store = 1` the `Store`
is a name, not a keyword.

`print` and `println` are intrinsics, not keywords, and concrete UI components
such as `Text`, `Button`, and `VStack` are neither keywords nor reserved: they
are ordinary names a program imports.

## Reserved Words

These words are reserved for future use. They have no syntax in this
specification, yet they MUST NOT be used as identifiers — as variables,
functions, types, fields, type parameters, or members — and a program that
does so is ill-formed. Reserving them leaves room for later syntax.

```text
abstract actor arguments associatedtype bench
case cfg channel class debugger default defer
deinit delete derive do doc eval export
extension fallthrough final function generator
generic global guard init instanceof interface
isolated iterator lazy library local macro
meta module new open override package priv
protocol receiver ref rethrows select sender
static super switch task this typealias
unowned unsafe var virtual void weak with
yield
```

The list excludes every keyword: `move`, `self`, and `spawn` are keywords,
not reserved words. `_` is neither; it is the wildcard name.

## Literals

A literal denotes a value directly. Spellings are given here; the resulting
types in [types/primitive-types.md](types/primitive-types.md) and the typing
of each literal form in [expressions/literals.md](expressions/literals.md).

### Numbers

```text
NumberLiteral  ::= IntegerLiteral | FloatLiteral
IntegerLiteral ::= DecInt | HexInt | BinInt | OctInt
FloatLiteral   ::= DecInt '.' DecInt
DecInt   ::= Digit { Digit }
HexInt   ::= '0' ( 'x' | 'X' ) HexDigit { HexDigit }
BinInt   ::= '0' ( 'b' | 'B' ) BinDigit { BinDigit }
OctInt   ::= '0' ( 'o' | 'O' ) OctDigit { OctDigit }
Digit    ::= '0' .. '9'
HexDigit ::= Digit | 'a' .. 'f' | 'A' .. 'F'
BinDigit ::= '0' | '1'
OctDigit ::= '0' .. '7'
```

Examples: `42`, `3.14`, `0xff`, `0b1010`, `0o77`.

- There are no digit separators (`1_000`), no exponents (`1e5`), no suffixes
  (`42u8`), no imaginary literals. Once a literal's form is consumed, an
  immediately following ASCII letter or `_` makes it invalid: `1a`, `1e5`, and
  `0xfg` are errors.
- A `.` belongs to a float literal only when a digit follows it: in `1...5`
  the dots are the range operator, and in `1.foo` the number is `1`. It is
  also withheld when the number begins immediately after a member-access
  token: in `p.0.1` the tokens are `p` `.` `0` `.` `1` — two chained
  positional accesses, never the float `0.1`.

### Strings

```text
StringLiteral ::= '"' { StringChar | EscapeSeq } '"'
                | "'" { StringChar | EscapeSeq } "'"
StringChar    ::= <any character except a line terminator, quote, or backslash>
EscapeSeq     ::= '\' ( '"' | "'" | '\' | 'n' | 't' | 'r' )
               | '\u' HexDigit HexDigit HexDigit HexDigit
               | '\u' '{' HexDigit { HexDigit } '}'
```

`"` and `'` are interchangeable delimiters. A string literal MUST NOT contain
a raw line terminator, an unclosed one is a lexical error, and neither
delimiter interpolates: `${...}` inside `"..."` or `'...'` is literal text.

The escapes are `\"` and `\'` (the quote character), `\\` (backslash), `\n`,
`\t`, `\r` (line feed, tab, carriage return), `\uXXXX` (exactly four hex
digits), and `\u{...}` (one to six hex digits in braces). A `\u` escape MUST
name a valid Unicode scalar value — at most U+10FFFF and never a surrogate in
U+D800–U+DFFF. Any other escape sequence is a lexical error.

### Template Literals

```text
TemplateLiteral ::= '`' { TemplateText | EscapeSeq | Interpolation } '`'
Interpolation   ::= '${' { <tokens> } '}'
```

- Templates are delimited by backticks and MAY span lines; an unterminated
  one is a lexical error.
- `${expr}` interpolates: the material between `${` and the matching `}` is
  tokenized as ordinary source with braces counted, so calls, operators, and
  nested templates all work inside an interpolation.
- In template text `` \` `` produces a backtick and `\$` a literal `$`, so
  `` `${not interpolation}` `` can be written as text; string escapes are
  accepted too. Interpolable operand types are given in
  [expressions/literals.md](expressions/literals.md).

### Booleans and Null

`true`, `false`, and `null` are keywords whose spelling is a literal: the
first two are boolean literals, `null` the null literal forming optional
types, and [types/primitive-types.md](types/primitive-types.md) gives their
types.

## Operators and Delimiters

The complete inventory of operator and delimiter tokens is below; the word
operators `and` and `or` are tokens too, though spelled as identifiers, and
precedence is defined in [expressions/operators.md](expressions/operators.md).

| Category | Tokens | Reading |
|----------|--------|---------|
| Grouping | `(` `)` `{` `}` `[` `]` | Call, block, list/map literal, subscript |
| Separators | `,` `;` | Item separator; optional statement terminator |
| Type and name separator | `:` `::` | Type/attribute label; enum variant path |
| Member access | `.` `?.` | Field, method, namespace member, tuple position; optional member |
| Assignment | `=` `:=` | Assignment; sugar for a mutable binding |
| Comparison | `==` `!=` `<` `>` `<=` `>=` | Equality and relational tests |
| Range and spread | `..<` `...` | Half-open and closed ranges; spread inside a literal |
| Arithmetic | `+` `-` `*` `/` `%` `**` | Additive, multiplicative, modulo, power |
| Bitwise and shift | `&` `\|` `^` `~` `<<` `>>` | And, or, xor, complement, shifts |
| Logical and conditional | `!` `?` `??` `and` `or` | Negation, ternary `? :` and postfix propagation, nullish coalescing, connectives |
| Arrow | `=>` | Match arm body; closure body |
| UI and state | `@` `$` | State declarations; two-way binding prefix |
| Reserved symbol | `#` | Not part of the language; it MUST NOT appear in source |

`::` names an enum variant and nothing else (`Theme::Dark`); `.` reaches
fields, methods, and namespace members (`Theme.value`, `Math.max`). `:=` is
sugar for `let mut`; see
[statements/let-and-assignment.md](statements/let-and-assignment.md).

## Longest Match

At each position the lexer MUST produce the longest token in the inventory
that matches; a shorter one is taken only when nothing longer matches, and a
position where no token matches is a lexical error.

| Source | Tokens | Reading |
|--------|--------|---------|
| `a...b` | `a` `...` `b` | `...` as the closed-range operator between operands |
| `...b` | `...` `b` | `...` as a spread prefix inside a literal |
| `x?.y` | `x` `?.` `y` | Optional member access, not `?` followed by `.` |
| `[1...5]` | `[` `1` `...` `5` `]` | One element: a closed range, not a spread |
| `[...xs]` | `[` `...` `xs` `]` | One element: a spread of `xs` |
| `iffy` | `iffy` | One identifier; `if` is not split off |
| `a..b` | `a` `.` `.` `b` | Two `.` tokens; `..` is not a token |
| `p.0.1` | `p` `.` `0` `.` `1` | Two chained positional accesses, not the float `0.1` |
| `p?.0` | `p` `?.` `0` | Optional positional access; `?` joins the following `.` |
| `a??b` | `a` `??` `b` | Nullish coalescing, not two `?` marks |
| `a? ?? b` | `a` `?` `??` `b` | Postfix propagation followed by `??` |

Because `..` does not exist, `..<` and `...` are always taken whole, and `?`
only when not followed by `.` or `?`: `p?.y` and `a??b` each form one token,
and separating the marks, as in `a? ?? b`, is how the postfix propagation
operator is written next to `??`. Whether `...` spreads or closes a range
is decided by position, not by lexing: inside a list literal or a brace map an
element beginning with `...` is a spread, while a `...` between two operands
is the closed-range operator. Both emit the same token.

## Conformance

A source file is lexically conforming when these rules produce a token stream
that reaches end of input without a lexical error. The lexical errors are: a
character that cannot start any token (including `#` anywhere, and a BOM away
from the start), an unterminated block comment, string, or template, an
invalid escape sequence, and an invalid number literal.

Keywords, reserved words, and wildcards are produced as their own tokens
here; using one where an identifier is required is rejected afterwards — by
the parser, and for `_` by the binding rules of [names.md](names.md).
