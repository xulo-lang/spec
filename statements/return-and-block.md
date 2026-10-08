# Return and Blocks

A block is both a scope and a value: `{ stmt; stmt; expr }` evaluates to the
value of its final expression where a value is expected, and `return` leaves
a function with an explicit value. This chapter defines block typing, the
rules of `return`, implicit return, and the evaluation order of statements.
Binding forms are specified in
[`let-and-assignment.md`](let-and-assignment.md); the statement-value rule
that constrains expression statements is in [`README.md`](README.md).

## Blocks as expressions

```xulo
let x = {
  let a = 1
  let b = 2
  a + b            // the value of the block is 3
}
```

- A block is written `{ stmt; stmt; …; expr }`. Its **value** is the value
  of its final expression, and that value is produced only where an
  expression is expected — as a binding initializer, an operand, an
  argument, a `return` operand, an arm body, or the trailing expression of
  an enclosing block.
- A block with no final expression has type `unit`.
- **The final expression MUST NOT be followed by `;` if its value is
  wanted.** A trailing `;` turns the expression into an expression
  statement, which discards the value; for a non-`unit` expression that is
  itself an error under the statement-value rule
  ([`README.md`](README.md)).

```xulo
fn f(): int {
  42           // legal: 42 is the trailing expression
}

fn g(): int {
  42;          // error: the trailing `;` discards the value
}

fn h() {
  print("hi")  // legal: the trailing expression has type unit
}
```

- Blocks nest without limit: the final expression of an outer block MAY be an
  inner block.

```xulo
let y = {
  let a = 1
  {
    let b = 2
    a + b        // inner block: value 3, which is the outer block's value
  }
}
```

- Each statement inside a block introduces a scope; bindings are visible
  from their declaration to the end of the block
  ([`../names.md`](../names.md)).

## Block typing

The type of a block is the type of its final expression, or `unit` when the
block has none. When a block is the body of a function, that rule becomes
the return rule:

- the final expression of a function body is the function's return value,
  unless a `return` statement executes first;
- a body whose final expression has type `unit` returns `unit`;
- when the function declares a return type, the final expression MUST have
  that type; a mismatch is a compile-time error
  ([`../type-system/errors.md`](../type-system/errors.md)).

## `return`

```xulo
fn subtract(a: int, b: int): int {
  return a - b
}

fn log(message: string) {
  print(message)
  return            // legal: the declared return type is unit
}
```

- The form is `return expr` or `return`.
- The type of `expr` MUST match the declared return type of the enclosing
  function; a mismatch is a compile-time error.
- `return` without an expression is valid **only** when the declared return
  type is `unit`. Writing a bare `return` in a function that declares any
  other return type is an error.
- `return` outside a function or closure body is a compile-time error.
- `return` MAY appear inside a final expression. When the executed branch
  runs it, the function returns immediately; the rest of the enclosing
  expression is not evaluated:

```xulo
fn classify(n: int): int {
  let doubled = if n < 0 { return 0 } else { n * 2 }
  doubled + 1
}
```

  `classify(-3)` runs `return 0` in the then-branch and the function
  evaluates to `0`; `classify(4)` takes the else-branch, binds `doubled` to
  `8`, and returns `9` through the trailing expression `doubled + 1`. The
  same pattern is available inside `match` arms, loop bodies, and nested
  blocks, because `return` leaves the innermost enclosing function no matter
  how deeply it is nested.

## Implicit return

```xulo
fn add(a: int, b: int): int {
  a + b            // trailing expression: the return value
}

fn greet(name: string) {
  print("hello " + name)   // trailing expression is unit
}

fn bad(): int {
  print("x")       // error: trailing expression is unit, declared return is int
}
```

- The trailing expression of a function body is the function's return value;
  this is implicit return. It is equivalent to writing `return` before that
  expression.
- A body that ends in a statement — a binding, an assignment, a loop — has
  no trailing expression and returns `unit`.
- When a return type is declared, the trailing expression MUST match it.
  When no return type is declared, the function returns `unit`, and the
  trailing expression, if any, MUST have type `unit`.

## Early exit patterns

Guard-style `return`s at the top of a body handle the exceptional cases
first, leaving the common path as the trailing expression:

```xulo
fn clamp(n: int, lo: int, hi: int): int {
  if lo > hi { return lo }
  if n < lo { return lo }
  if n > hi { return hi }
  n                  // common case: implicit return
}
```

- Each guard is an `if` with a `boolean` condition whose then-branch is a
  `return`; the guards are statements, so their values are discarded.
- The pattern keeps the common path short and makes the exceptional path
  explicit. It is a convention, not a special form: any mix of `return` and
  a trailing expression is well-formed when the types agree.

## Blocks in other contexts

A block is accepted wherever the corresponding construct accepts a body:

| Context | Body block | Notes |
|---------|------------|-------|
| `if` / `else if` / `else` | `{ … }` | branch value is the block's value ([`../expressions/control-flow.md`](../expressions/control-flow.md)) |
| `match` arm | `{ … }` or a bare expression | arm body ([`../expressions/control-flow.md`](../expressions/control-flow.md)) |
| `for` / `while` | `{ … }` | loop body; the loop itself has type `unit` |
| Function body | `{ … }` | final expression is the return value |
| Closure body | `{ … }` or a bare expression | specified in [`../expressions/closures.md`](../expressions/closures.md) |
| Component children | `{ … }` | trailing block of a component invocation ([`../components/view-syntax.md`](../components/view-syntax.md)) |

## Evaluation of statements

Statements in a block are evaluated **sequentially**, in source order, from
first to last. Each statement completes before the next begins; there is no
reordering of side effects across statements. Evaluation order inside a
single expression — operands, arguments, assignment places — is specified in
[`../expressions/README.md`](../expressions/README.md). A `return`, `break`,
or `continue` abandons the remaining statements of the current path
immediately.
