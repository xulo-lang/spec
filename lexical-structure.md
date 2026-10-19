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

The language has exactly the following closed set of 58 keywords; every other
word is an ordinary identifier unless it is reserved below. No keyword MAY be
used as one.

| Category | Keyword | Purpose |
|----------|---------|---------|
| Declarations | `const` | Declares a compile-time constant |
| | `enum` | Declares an enumeration type |
| | `fn` | Declares a function or a function-typed value |
| | `impl` | Declares the implementation of a trait for a type |
| | `let` | Declares a binding; immutable unless preceded by `mut` |
| | `mod` | Declares a module — a file (`mod form;`) or a block (`mod x { … }`) |
| | `mut` | Marks a binding or a borrow as mutable |
| | `self` | The receiver of a method inside an `impl` block; in a path, the current module |
| | `struct` | Declares a structure type |
| | `trait` | Declares a trait |
| | `type` | Declares a type alias |
| | `where` | Introduces a generic constraint clause after a signature |
| Control flow | `break` | Leaves the innermost enclosing `for` or `while` |
| | `continue` | Starts the next iteration of the innermost loop |
| | `defer` | Registers an expression to run when the enclosing block exits |
| | `else` | Introduces the alternative of an `if` |
| | `for` | Declares a counting or iterating loop |
| | `if` | Conditional expression or statement |
| | `in` | Separates the loop variable from the iterable in `for` |
| | `match` | Pattern matching over a scrutinee |
| | `panic` | Stops the program with an unrecoverable error |
| | `return` | Returns a value from the enclosing function |
| | `while` | Declares a condition-controlled loop |
| Modules | `as` | Renames a used or re-exported name |
| | `pub` | Marks a declaration, member, module, or use as publicly visible |
| | `super` | In a path, the parent module of the current one |
| | `use` | Brings names of a path into scope — singly, grouped, or by glob |
| Async and concurrency | `async` | Declares an asynchronous function or closure |
| | `await` | Unwraps a `Task<T>` inside an `async` body |
| | `lock` | Enters a critical section over `shared` state |
| | `shared` | Declares state that may be shared between tasks |
| | `spawn` | Starts a task, thread, process, or pooled job |
| Literals and operators | `and` | Logical conjunction (word operator) |
| | `false` | The `bool` literal for falsity |
| | `is` | Type test (word operator) |
| | `not` | Negated type test (word operator) |
| | `or` | Logical disjunction (word operator) |
| | `true` | The `bool` literal for truth |
| Ownership | `copy` | Evaluates an expression and deep-copies the result |
| | `move` | Transfers ownership of a value out of its binding |
| Primitive types | `bool` | The two-valued boolean type |
| | `string` | The immutable UTF-8 text type |
| | `number` | The numeric union type `int \| float` |
| | `int` | 64-bit signed integer type |
| | `float` | 64-bit IEEE-754 binary64 type |
| | `i8` | 8-bit signed integer type |
| | `i16` | 16-bit signed integer type |
| | `i32` | 32-bit signed integer type |
| | `i64` | 64-bit signed integer type |
| | `u8` | 8-bit unsigned integer type |
| | `u16` | 16-bit unsigned integer type |
| | `u32` | 32-bit unsigned integer type |
| | `u64` | 64-bit unsigned integer type |
| | `f32` | 32-bit IEEE-754 binary32 type |
| | `f64` | 64-bit IEEE-754 binary64 type |
| | `null` | The null literal; the type of `null` |
| | `unit` | The unit type |
| | `unknown` | The top type |

`from` and `import` are no longer keywords: with string specifiers and the
side-effect form gone, no import mechanism remains to spell. Neither is
reserved — both are ordinary identifiers.

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
case cfg channel class debugger default
deinit delete derive do doc eval export
extension fallthrough final function generator
generic global guard init instanceof interface
isolated iterator lazy library local macro
meta module new open override package priv
protocol receiver ref rethrows select sender
static switch task this typealias
unowned unsafe var virtual void weak with
yield
```

The list excludes every keyword: `mod`, `move`, `self`, `spawn`, and `super`
are keywords, not reserved words. `_` is neither; it is the wildcard name.

## Operators and Delimiters

The complete inventory of operator and delimiter tokens is below; the word
operators `and`, `or`, `is`, and `not` are tokens too, though spelled as
identifiers, and precedence is defined in
[expressions/operators.md](expressions/operators.md).

| Category | Tokens | Reading |
|----------|--------|---------|
| Grouping | `(` `)` `{` `}` `[` `]` | Call, block, list/map literal, subscript |
| Separators | `,` `;` | Item separator; optional statement terminator |
| Type and name separator | `:` `::` | Type/attribute label; path separator — packages, modules, variants, associated names |
| Member access | `.` `?.` | Field, method, map entry, tuple position, on a value; optional member |
| Assignment | `=` `:=` | Assignment; sugar for a mutable binding |
| Comparison | `==` `!=` `<` `>` `<=` `>=` | Equality and relational tests |
| Type test | `is` `not` | Type test and its negation |
| Range and spread | `..<` `...` | Half-open and closed ranges; spread inside a literal |
| Arithmetic | `+` `-` `*` `/` `%` `**` | Additive, multiplicative, modulo, power |
| Bitwise and shift | `&` `\|` `^` `~` `<<` `>>` | And, or, xor, complement, shifts |
| Logical and conditional | `!` `?` `??` `and` `or` | Negation, ternary `? :` and postfix propagation, nullish coalescing, connectives |
| Arrow | `=>` | Match arm body; closure body |
| UI and state | `@` `$` | State declarations; two-way binding prefix |
| Attributes | `#` | Declaration attribute marker |

`test` is **not** added to the reserved words: it becomes an ordinary
identifier — usable as a variable name, a function name, a field, anywhere an
identifier is allowed — exactly as `from` and `import` did when the import
syntax was retired. The old spelling `test fn …` no longer parses as a
declaration; it is a syntax error, never a reinterpretation.

`::` names every qualified name — a path (`ui::hstack::HStack`), a variant
(`Theme::Dark`), an associated or namespace name (`Math::PI`) — and nothing
else. `.` reaches into a value: fields, methods, map member entries, tuple
positions (`user.name`, `r.area()`, `p.0`); it never names a namespace or a
module member, which use `::`. `:=` is sugar for `let mut`; see
[statements/let-and-assignment.md](statements/let-and-assignment.md).

A package segment in a path is written with its hyphens unwspaced
(`lsp-server`); in an expression or type position the chain is part of a path
only when `::` follows it, so `lsp-server::run()` is a call and
`total - Math::PI` is subtraction
([path-and-access.md](expressions/path-and-access.md)).
## Longest Match

At each position the lexer MUST produce the longest token in the inventory
that matches; a shorter one is taken only when nothing longer matches, and a
position where no token matches is a lexical error, which carries no
diagnostic code ([type-system/errors.md](type-system/errors.md)).

| Source | Tokens | Reading |
|--------|--------|---------|
| `a...b` | `a` `...` `b` | `...` as the closed-range operator between operands |
| `...b` | `...` `b` | `...` as a spread prefix inside a literal |
| `x?.y` | `x` `?.` `y` | Optional member access, not `?` followed by `.` |
| `[1...5]` | `[` `1` `...` `5` `]` | One element: a closed range, not a spread |
| `[...xs]` | `[` `...` `xs` `]` | One element: a spread of `xs` |
| `iffy` | `iffy` | One identifier; `if` is not split off |
| `isnot` | `isnot` | One identifier; `is` is not split off |
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
character that cannot start any token (and a BOM away from the start), an
unterminated block comment, string, or template, an invalid escape sequence,
and an invalid number literal.

Keywords, reserved words, and wildcards are produced as their own tokens
here; using one where an identifier is required is rejected afterwards — by
the parser, and for `_` by the binding rules of [names.md](names.md).
