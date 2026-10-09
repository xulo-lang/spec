# Expression Statements

An expression statement is a bare expression written where a statement is
expected. It is evaluated once, for its effects; its value is discarded — and
discarding a value is exactly what the statement-value rule polices. The rule
is stated once in [`README.md`](README.md), quoted again below; this chapter
explains when it bites, what it excepts, and the structural restrictions that
keep statement position unambiguous. Blocks, `return`, and the trailing
expression of a block are specified in [`return-and-block.md`](return-and-block.md).

## The statement-value rule

> An expression statement that is not `unit` is an error, EXCEPT when it is a
> control-flow construct (`if`, `match`, `for`, `while`) whose value is
> discarded.

The rule exists because a computed value that nobody reads is almost always a
mistake: a comparison that was meant to be a call, a call whose result was
meant to initialize a binding, an arithmetic expression left over from an
edit. Effects are the reason to write an expression as a statement; anything
the expression computes beyond `unit` is required to be used.

```xulo
let mut xs = [1, 2, 3]
let mut total = 0

xs.sort()            // legal: sort returns unit
total + 1            // error: value discarded (E0218)
if total > 0 { print("positive") } else { print("zero") }   // legal
total > 0 ? a() : b() // error: ternary is not a control-flow construct
```

In expression position — a `let` initializer, an operand, an argument, a
`return` operand, a match arm body, or a block's trailing expression — the
value is used and the rule does not apply:

```xulo
let sum = a + b      // legal: value initializes a binding
print(a + b)         // legal: value is an argument
```

## The exceptions

`if`, `match`, `for`, and `while` MAY stand in statement position
whatever their value is, because in that position the value of the branch,
arm, or body that runs is discarded. Their branches or arms MAY therefore
have unrelated types when the construct is a statement; in expression
position the common-type rules apply in full
([`../expressions/control-flow.md`](../expressions/control-flow.md)).

```xulo
if ready { prepare() } else { "not ready" }   // legal as a statement
match half(3) { Result::Ok(v) => str(v) Result::Err(e) => e }   // legal: value discarded
```

The exception covers only those four constructs. In particular a ternary
`c ? x : y` is not one of them ([`../expressions/operators.md`](../expressions/operators.md)),
and a `spawn` expression — though it names control flow — is deliberately not
one either: bind it, `await` it, or pass it on
([`../concurrency.md`](../concurrency.md)).

```xulo
spawn async { compute() }   // error: Task<T> discarded (E0218)
let t = spawn async { compute() }   // legal: the task is bound
```

## The leading `{` restriction

An expression statement MUST NOT begin with `{`: statement-position `{`
always introduces a block, so a construct that started with `{` would be read
as one. To use a map literal in statement-like position, parenthesize it
([`../expressions/literals.md`](../expressions/literals.md)):

```xulo
{ name: "lyy" }        // not a map literal: parsed as a block
({ name: "lyy" })      // legal: parenthesized map literal
{ print("hi") }        // a block, whose value is discarded under the rule
```

## Interaction with trailing expressions and `;`

A final expression in a block is the block's value; writing `;` after it
turns it into an expression statement, which discards the value — an error
for a non-`unit` value under the rule above
([`return-and-block.md`](return-and-block.md)). The converse holds too: an
expression statement separated by an optional `;` discards its value
regardless of what follows it on the line
([`README.md`](README.md#the-statementexpression-split)).

## Evaluation

Expression statements are evaluated in the order they appear in the block;
each expression is evaluated exactly once, with the side effects its
subexpressions prescribe — a call performs its arguments' evaluation before
the call itself, an assignment evaluates its right-hand side and its place
subexpressions before the write ([`let-and-assignment.md`](let-and-assignment.md)).
No reordering across statement boundaries is permitted by this chapter.

## Errors

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0218` | an expression statement has a type other than `unit` and is not an excepted control-flow construct | `a + b` with `a: int` | ``expected `unit`, found `int``` |
| `E0402` | the statement is an assignment whose target is not a place | `f() = 2` | ``invalid assignment target`` |
| `W0103` | a statement can never run | a statement after `return` | ``unreachable code`` |

The codes are registered in [`../type-system/errors.md`](../type-system/errors.md);
the typing rules behind them are in
[`../type-system/checking-rules.md`](../type-system/checking-rules.md).

## Index

- [`README.md`](README.md) — the statement/expression split and termination.
- [`return-and-block.md`](return-and-block.md) — blocks as values, `return`,
  implicit return.
- [`../expressions/control-flow.md`](../expressions/control-flow.md) —
  `if`/`match` in both positions.
- [`../error-handling.md`](../error-handling.md) — `?` propagation and
  `panic` as expressions.
