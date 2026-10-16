# Lexical Structure — Delta: Keywords and the Lane Separators

> **Delta** — replaces the **Keywords** table and count, the **Reserved
> Words** list, and the separator rows and closing note of the **Operators
> and delimiters** table in `lexical-structure.md`.

## Keywords

The language has exactly the following closed set of 42 keywords; every other
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
| | `test` | Marks a function as a test: checked with the program, run only by a test tool |
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
| | `false` | The `Boolean` literal for falsity |
| | `is` | Type test (word operator) |
| | `not` | Negated type test (word operator) |
| | `null` | The null literal; an optional with no content |
| | `or` | Logical disjunction (word operator) |
| | `true` | The `Boolean` literal for truth |
| Ownership | `copy` | Evaluates an expression and deep-copies the result |
| | `move` | Transfers ownership of a value out of its binding |

`from` and `import` are no longer keywords: with string specifiers and the
side-effect form gone, no import mechanism remains to spell. Neither is
reserved — both are ordinary identifiers.

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

## Operators and delimiters

The token category table's separator rows change; the rest of the table is
unchanged:

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
| Reserved symbol | `#` | Not part of the language; it MUST NOT appear in source |

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
