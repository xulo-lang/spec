# Control Flow

Xulo is expression-oriented: `if` and `match` produce values, `for` and `while`
are statements of type `unit`, and `break`/`continue` jump to loop boundaries.
This chapter defines their syntax, typing, evaluation order, and scoping;
`try`/`catch` appears here only to redirect control and is specified in
[`error-handling.md`](../error-handling.md). The expression layer as a whole is
mapped in [`README.md`](README.md).

Two positions are distinguished throughout. An expression is in
**expression position** when its value is used: as a `let` initializer, an
operand, an argument, a `return` operand, an arm body, or the trailing
expression of a block. An expression is in **statement position** when it is
written as an expression statement (see
[`expression-statements.md`](../statements/expression-statements.md)). One rule
covers statement position everywhere in this specification:

> An expression statement that is not `unit` is an error, EXCEPT when it is a
> control-flow construct (`if`, `match`, `for`, `while`, `try`) whose value is
> discarded.

Accordingly, an `if` or `match` written as a statement MAY have branches or
arms of unrelated types, because the value of the branch that runs is
discarded. In expression position the typing rules below apply in full.

## `if` expressions

```xulo
let max = if a > b { a } else { b }

if condition {
  // ...
} else if other {
  // ...
} else {
  // ...
}
```

- The condition MUST have type `boolean`. There is no truthiness: any other
  condition type is a compile-time error
  ([`checking-rules.md`](../type-system/checking-rules.md)).
- `else if` is not a separate construct: it is an `if` used as the `else`
  branch and may be chained without limit.
- The condition is evaluated once; exactly one branch is evaluated.
- `if` is an expression. Its type is determined as follows:
  - with an `else` branch: both branches MUST have a common type
    ([`checking-rules.md`](../type-system/checking-rules.md)) and the `if` has
    that common type; branches with no common type are a compile-time error
    ([`errors.md`](../type-system/errors.md));
  - without an `else` branch: the type is `unit`, and the value of the then
    branch, if any, is discarded. In particular, such an `if` cannot supply a
    value of any type other than `unit`.
- In statement position the rule quoted in the introduction applies.

```xulo
// expression position: both branches are `string`, so the type is `string`
let parity = if n % 2 == 0 { "even" } else { "odd" }
```

## `match` expressions

`match` tests a value against patterns and runs the body of the first arm that
matches.

```xulo
let label = match x {
  0 => "zero"
  1 => "one"
  _ => "many"
}
```

- The scrutinee is evaluated exactly once, before any arm is tested, no matter
  how many arms are tried.
- Arms are tested in source order; the first arm whose pattern matches wins and
  its body is evaluated. All later arms are skipped.
- An arm is `pattern => expression` or `pattern => { block }`; arms are
  separated by newlines or optional commas.
- `match` is an expression. In expression position all arm bodies MUST have a
  common type ([`checking-rules.md`](../type-system/checking-rules.md)); arms
  with no common type are a compile-time error
  ([`errors.md`](../type-system/errors.md)). In statement position the rule
  quoted in the introduction applies and the arm values are discarded.

```xulo
match score {          // statement position: the arm values are discarded
  0 => "zero"
  _ => { print("scored") }
}
```

### Patterns

| Pattern | Form | Matches |
|---------|------|---------|
| Literal | `0`, `3.5`, `"ok"`, `true`, `null` | values equal to the literal ([`literals.md`](literals.md)) |
| Wildcard | `_` | any value; binds nothing |
| Binding | `x` | any value; binds it to `x` for the body of the arm |
| Variant | `Enum::Variant` | a payload-less variant |
| Variant with payload | `Enum::Variant(p1, ..., pn)` | that variant; one sub-pattern per payload field, in declaration order |
| Struct deconstruction | `Type(p1, ..., pn)` | a value of that struct type; one sub-pattern per field, in declaration order |
| Range | `a..<b`, `a...b` | numeric values inside the half-open or closed range |

- Enum variants are written with `::` in patterns and in expressions
  (`Shape::Circle(r)`), never with `.`.
- Sub-patterns nest arbitrarily: `Shape::Dot(Point(x, y))` matches a variant
  whose payload is a `Point` and then deconstructs that `Point`.
- Named payload fields are still matched positionally: `Shape::Rect(w, h)`
  matches a `Rect(width: int, height: int)` variant in declaration order.
- A payload or field position that is not bound MUST be written `_`:
  `Shape::Circle(_)` matches any radius without binding it.
- The operands of a range pattern MUST be numeric literals of the same numeric
  type as the scrutinee. `0...9` includes both endpoints; `0..<10` excludes the
  upper endpoint.
- Every pattern MUST be compatible with the type of the scrutinee; otherwise the
  `match` is a compile-time error.

```xulo
struct Point { x: int, y: int }

enum Shape {
  Dot(Point)
  Rect(int, int)
}

fn describe(s: Shape): string {
  match s {
    Shape::Dot(Point(x, y)) => `point ${x},${y}`
    Shape::Rect(w, h) => `${w}x${h}`
  }
}
```

### Exhaustiveness and reachability

- A `match` over an enum MUST cover every variant of that enum, either with a
  variant pattern (possibly nested inside another pattern) or with a wildcard
  or binding arm.
- A `match` over `boolean` MUST cover `true` and `false`, or have a wildcard or
  binding arm.
- For every other scrutinee type, the arms MUST include a wildcard or binding
  arm, because literal and range patterns cannot cover the whole type.
- A `match` that fails these requirements is a compile-time error
  ([`errors.md`](../type-system/errors.md)).
- An arm that can never be selected is an error. In particular, every arm
  after a wildcard or binding arm is unreachable, and a literal that repeats an
  earlier literal arm is unreachable.

```xulo
fn describe(flag: boolean): string {
  match flag {
    true => "yes"
    false => "no"
  }
}

let grade = match score {
  90...100 => "A"
  80..<90 => "B"
  _ => "F"
}

match n {
  _ => "any"
  0 => "zero"   // error: unreachable arm
}
```

## `for` loops

```xulo
for item in items {
  print(item)
}
```

- The form is `for x in expression { ... }`. The iterable expression is
  evaluated exactly once, before the first iteration, and the loop then visits
  the sequence of elements produced by that evaluation. Assignments to the
  source binding, or mutations of its contents, performed after the loop has
  begun do not affect the iteration in progress: the elements to be visited are
  fixed when the loop begins.
- The iterable type MUST be one of the following; any other type is a
  compile-time error ([`errors.md`](../type-system/errors.md)).

| Iterable type | Loop binding | Iteration order |
|---------------|--------------|-----------------|
| `list<T>` | `T` | index order |
| `map<K, V>` | `K` | insertion order of keys |
| `set<T>` | `T` | unspecified; programs MUST NOT depend on one |
| `Range<T>` | `T` | ascending from `start` |

- Iterating a `map` visits its **keys**; the corresponding values are read with
  a subscript (see [`path-and-access.md`](path-and-access.md)). Iterating a
  `list`, `map`, or `Range` is core syntax; the other operations on `map` and
  `set` are prelude functions (see
  [`../types/composite-types.md`](../types/composite-types.md)).
- The loop variable is a fresh **immutable** binding for every iteration. It
  MAY shadow an outer binding of the same name, MUST NOT be assigned, and is
  not visible after the loop. To accumulate across iterations, use a `let mut`
  binding declared outside the loop. Because each iteration creates a distinct
  binding, a closure created during an iteration captures that iteration's
  binding (see [`closures.md`](closures.md)).
- The value of a `for` loop is `unit`.

```xulo
fn sum(xs: list<int>): int {
  let mut acc = 0
  for x in xs {
    acc = acc + x
  }
  acc
}
```

## Ranges

- `a..<b` is **half-open**: it includes `a` and excludes `b`.
- `a...b` is **closed**: it includes both `a` and `b` (Swift semantics).
- Both operands MUST have the same numeric type; the result is a value of the
  built-in generic type `Range<T>`, which MAY be bound and iterated later.
- Iterating a range yields `start`, `start + 1`, and so on while the value has
  not passed the end bound.

| Expression | Iteration values |
|------------|------------------|
| `0..<3` | `0`, `1`, `2` |
| `0...3` | `0`, `1`, `2`, `3` |
| `1...1` | `1` |
| `2..<2` | none |
| `5..<1` | none |
| `5...1` | none |

- A range whose `start` is greater than its `end` iterates zero times. There is
  no reverse iteration: neither `..<` nor `...` descends.
- `..<` and `...` are non-associative operators at the range level of the
  precedence table in [`operators.md`](operators.md).
- Inside a list or object literal an element that starts with `...` is a spread
  of that literal ([`literals.md`](literals.md)); in every other position `...`
  is the closed-range operator.

## `while` loops

```xulo
let mut count = 0
while count < 10 {
  count = count + 1
}
```

- The condition is evaluated before every iteration and MUST have type
  `boolean`; there is no truthiness
  ([`checking-rules.md`](../type-system/checking-rules.md)).
- The body runs zero or more times; `while` is a statement and its value is
  `unit`.

## `break` and `continue`

- `break` and `continue` are statements, valid inside the body of a `for` or a
  `while` loop, including inside `if` or `match` arms nested in that body.
- `break` terminates the innermost enclosing loop; `continue` abandons the
  current iteration and proceeds to the next one — for `for` loops to the next
  element, for `while` loops to a fresh evaluation of the condition.
- `break` MUST NOT carry a value. There is no `loop` expression and no loop
  labels, so both statements always target the innermost enclosing loop.
- `break` or `continue` outside a loop, or inside a function or closure body
  nested within a loop, is a compile-time error
  ([`errors.md`](../type-system/errors.md)).

```xulo
for row in 0..<3 {
  for col in 0..<3 {
    if col == 1 { continue }   // skips the rest of this inner iteration
    if row == 2 { break }      // leaves the inner loop only
    print(`${row},${col}`)
  }
}
```

## `try`/`catch` as control flow

`throw expr` abandons the current control path and transfers control to the
nearest enclosing handler; `try { ... } catch e { ... }`,
`try { ... } catch e: T { ... }`, and `finally { ... }` receive it. A loop does
not catch anything by itself: an uncaught throw leaves the loop.

```xulo
try {
  let total = parseTotal(text)
  print(total)
} catch e: FormatError {
  print("invalid input")
} finally {
  print("done")
}
```

The typing of handlers, rethrowing, and the value of a `try` construct are
specified in [`error-handling.md`](../error-handling.md).

## Scoping rules

- Every block body — an `if` branch, a `match` arm, a loop body, and a
  `try`/`catch`/`finally` block — opens a new block scope: a binding introduced
  inside is visible only inside that block; see [`names.md`](../names.md).
- Shadowing an outer binding of the same name inside a block is allowed; the
  outer binding is visible again after the block.
- The loop variable of a `for` loop lives in the scope of a single iteration,
  so nested loops and repeated iterations never share a loop binding.

```xulo
if true {
  let note = "inside"
  print(note)
}
print(note)   // error: `note` is not in scope here
```

## Evaluation order and short-circuit summary

Operands of binary operators are evaluated left to right (see
[`operators.md`](operators.md)). The constructs of this chapter evaluate as
follows:

| Construct | Evaluated | Skipped |
|-----------|-----------|---------|
| `A and B` | `A` first; `B` only when `A` is `true` | `B` when `A` is `false` |
| `A or B` | `A` first; `B` only when `A` is `false` | `B` when `A` is `true` |
| `c ? x : y` | `c`, then exactly one of `x`, `y` | the branch not selected |
| `a ?? b` | `a`; `b` only when `a` is `null` | `b` when `a` is not `null` |
| `if c { A } else { B }` | `c` once, then the selected branch | the other branch |
| `match e { ... }` | `e` once, then patterns top to bottom, then the body of the first match | every later arm |
| `while c { B }` | `c` before each iteration, then `B` while `c` is `true` | `B` when `c` is `false` |
| `for x in it { B }` | `it` once, then one element binding and `B` per element | the whole body when the sequence is empty |
| `break` / `continue` | the jump itself | the rest of the current iteration; `break` also skips the rest of the loop |

`and` and `or` are word operators and short-circuit: the right operand may be
skipped entirely, so side effects placed in it do not necessarily happen.
