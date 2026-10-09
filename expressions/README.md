# Expressions

Xulo is expression-oriented: nearly every construct evaluates to a value, and
programs are built by composing expressions rather than by sequencing
instructions. Literals, operators, calls, closures, `if`, `match`, blocks, and
`await` all produce values. Constructs that exist only to introduce a name or
to effect a change produce no meaningful value; where such a construct must be
given a type, that type is `unit` (see
[`../types/primitive-types.md`](../types/primitive-types.md)).

## Expressions versus Statements

The split is exact: every construct is either an expression (it evaluates to a
value) or a statement (it does not). Statements are specified in
[`../statements/README.md`](../statements/README.md).

| Construct | Category |
|-----------|----------|
| `let x = e`, `let mut x = e`, `const K = e` | Statement |
| `place = value` | Statement |
| `return e`, `return` | Statement |
| `break`, `continue` | Statement |
| `for … in … { … }`, `while … { … }` | Statement |
| Expression statement (a bare expression in statement position) | Statement |
| `if` … `else` …, `match` …, block `{ … }` | Expression |
| Literals: integer, float, string, template, boolean, null, list, map, tuple | Expression |
| Operators, member access, subscript, calls | Expression |
| Closures, `await` | Expression |

`if`, `match`, and blocks written where a statement is expected are expression
statements: their value is computed and then discarded, and the constraints on
expression statements are specified in [`../statements/README.md`](../statements/README.md).
`break` and `continue` are never expressions; there is no `loop`
expression. Concurrency constructs (`spawn …`, `lock …`) are specified in
[`../concurrency.md`](../concurrency.md).

## Evaluation Order

Evaluation order is left-to-right and is normative: side effects happen in the
order stated here.

- The operands of a binary operator are evaluated left-to-right.
- The arguments of a call are evaluated left-to-right.
- The elements of a list literal, a map literal, or a tuple literal are
  evaluated left-to-right, spreads included.
- The scrutinee of `if` or `match` is evaluated before any arm is tested.

Three constructs short-circuit. `and` does not evaluate its right operand when
its left operand is `false`; `or` does not evaluate its right operand when its
left operand is `true`; `??` does not evaluate its right operand when its left
operand is not `null`. The ternary `?:` evaluates exactly one of its branches.

In an assignment the right-hand side is evaluated before the place is written;
the receiver and index subexpressions of the place are then evaluated
left-to-right, and only afterwards does the write occur.

## Precedence and Associativity

Operators bind from lowest to highest precedence as follows. This table is
normative and is reproduced in [`operators.md`](operators.md), which explains
each level; [`../grammar.md`](../grammar.md) encodes the same structure.

| Level | Operators | Assoc |
|-------|-----------|-------|
| assignment | `=` | right |
| ternary | `?:` | right |
| logical or | `or` | left |
| logical and | `and` | left |
| nullish | `??` | left |
| equality | `==` `!=` | left |
| relational | `<` `>` `<=` `>=` | non-assoc |
| range | `..<` `...` | non-assoc |
| bitwise or | `\|` | left |
| bitwise xor | `^` | left |
| bitwise and | `&` | left |
| shift | `<<` `>>` | left |
| additive | `+` `-` | left |
| multiplicative | `*` `/` `%` | left |
| power | `**` | right |
| unary | `!` `-` `~` `await` | prefix |
| postfix | `f(x)` `x[i]` `x.y` `x.0` `x?.y` `x?.0` `x?` | left |

Prefix spread `...expr` exists only inside list and map literals; it is not
part of the precedence chain. An element that begins with `...` is a spread,
otherwise `...` is the closed-range operator (see [`operators.md`](operators.md)).

## Place Expressions and Value Expressions

A **place expression** denotes a storage location that can be written: an
identifier bound by `let mut`, a field access whose receiver is a place, and a
subscript whose base is a place. A **value expression** is everything else; it
denotes a value, not a location.

Assignment and every other write target a place expression, and the place MUST
be backed by a mutable binding or a mutable field or borrow (see
[`../memory-and-runtime.md`](../memory-and-runtime.md)). Reading a place
expression — in any context other than as the target of an assignment — yields
its value, so a place can be used wherever a value of its type is expected.
Assigning to an identifier that is not backed by `mut` is a compile-time error.

## Index

- [`literals.md`](literals.md) — Integer, float, string, template, boolean, null, list, map, and tuple literals.
- [`operators.md`](operators.md) — Precedence, associativity, and the rules of every operator.
- [`path-and-access.md`](path-and-access.md) — Identifiers, member access, positional tuple access, `::` variant paths, subscripts, optional chaining.
- [`calls.md`](calls.md) — Call syntax, argument matching, method and generic calls, components.
- [`control-flow.md`](control-flow.md) — `if` and `match` as expressions, patterns and exhaustiveness, loops and `break`/`continue` as statements.
- [`closures.md`](closures.md) — Anonymous functions, capture, and function types.
- [`async-expressions.md`](async-expressions.md) — `async` bodies, `await`, and `Task<T>`.
