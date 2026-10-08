# Statements

Xulo is expression-oriented, but not purely so: a small set of constructs
exists only to introduce a name, to change a place, or to redirect control,
and those constructs are **statements** — they produce no meaningful value.
Everything else is an expression. This chapter defines the split, statement
termination, and the rule that constrains expression statements. The
expression layer as a whole is mapped in
[`../expressions/README.md`](../expressions/README.md).

## The Statement/Expression Split

The split is exact: every construct is either a statement or an expression.

| Construct | Category |
|-----------|----------|
| `let x = e`, `let mut x = e`, `let (a, b) = e`, `let { f } = e`, `const K = e` | Statement |
| `place = value` | Statement |
| `return e`, `return` | Statement |
| `break`, `continue` | Statement |
| `for … in … { … }`, `while … { … }` | Statement |
| Expression statement (a bare expression in statement position) | Statement |
| `if` … `else` …, `match` …, block `{ … }` | Expression |
| Literals, operators, member access, subscript, calls | Expression |
| Closures, `await` | Expression |

`break` and `continue` are statements valid only inside `for` and `while`
([`../expressions/control-flow.md`](../expressions/control-flow.md)). An
`if`, `match`, or block written where a statement is expected is an
expression statement: its value is computed and discarded under the rule
below. Failure propagation (`?`) and the unrecoverable `panic(...)` are
expressions of the error model
([`../error-handling.md`](../error-handling.md)).

## Statement Termination

The grammar is newline-insensitive: a statement ends when the next token
cannot continue it, and `;` is an **optional** statement terminator.
Newlines have no syntactic significance; this is not automatic semicolon
insertion. A `;` MAY be written after any statement and SHOULD be used to
separate several statements on one line. Nothing in the language requires
a `;`:

```xulo
let a = 1
let b = 2
print(a + b)

let c = 3; let d = 4; print(c + d)   // semicolons separate one-line statements
```

## The Statement-Value Rule

> An expression statement that is not `unit` is an error, EXCEPT when it is a
> control-flow construct (`if`, `match`, `for`, `while`) whose value is
> discarded.

The exception covers control-flow constructs in statement position: their
value is discarded, so branches or arms MAY have unrelated types. The full
treatment — including the leading-`{` restriction below and the error code —
is in [`expression-statements.md`](expression-statements.md).

```xulo
if c { print("a") } else { print("b") }   // legal: value discarded
match half(3) { Result::Ok(v) => str(v) Result::Err(e) => e }  // legal: value discarded
a + b              // error: not unit, not a control-flow construct
print("hi")        // legal: unit-returning call
let x = a + b      // legal in expression position: value initializes x
```

An expression statement MUST NOT begin with `{`, because such a construct
would be read as a block; write an object literal in parentheses when it
must appear in statement-like position.

## Blocks and Scope

A block `{ … }` is both a value and a scope: its type is the type of its
final expression (or `unit` when there is none), and every statement inside
introduces bindings visible from their declaration to the end of the block.
Blocks, returns, and implicit return are specified in
[`return-and-block.md`](return-and-block.md); scoping and shadowing in
[`../names.md`](../names.md).

## Index

- [`let-and-assignment.md`](let-and-assignment.md) — `let`, `let mut`, `const`, assignment, destructuring, and shadowing.
- [`return-and-block.md`](return-and-block.md) — blocks as expressions, `return`, implicit return, and statement evaluation order.
- [`expression-statements.md`](expression-statements.md) — bare expressions in statement position and the constraints on them.
