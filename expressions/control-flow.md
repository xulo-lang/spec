# Control Flow

Xulo is expression-oriented: `if` and `match` produce values, `for` and `while`
are statements of type `unit`, and `break`/`continue` jump to loop boundaries.
This chapter defines their syntax, typing, evaluation order, and scoping;
failure propagation and `panic` are specified in
[`error-handling.md`](../error-handling.md). The expression layer as a whole is
mapped in [`README.md`](README.md).

Two positions are distinguished throughout. An expression is in
**expression position** when its value is used: as a `let` initializer, an
operand, an argument, a `return` operand, an arm body, or the trailing
expression of a block. An expression is in **statement position** when it is
written as an expression statement (see
[`expression-statements.md`](../statements/expression-statements.md)). One rule
covers statement position everywhere in this specification:

> An expression statement that is not `unit` is an error (`E0218`), EXCEPT when
> it is a control-flow construct (`if`, `match`, `for`, `while`) whose value is
> discarded, or a `send` whose `Task<R>` is discarded.

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

- The condition MUST have type `bool`. There is no truthiness: any other
  condition type is a compile-time error (`E0201`)
  ([`checking-rules.md`](../type-system/checking-rules.md)).
- `else if` is not a separate construct: it is an `if` used as the `else`
  branch and may be chained without limit.
- When the condition of an `if` — directly, or the condition wrapped in
  parentheses — is an **is-expression** `x is T` or `x is not T`, where `x` is
  an identifier of type `S` and `T` is a testable type assignable to `S`, the
  branches check with `x` narrowed:
  - with `x is T`: the then branch checks with `x : T`, and the else branch
    checks with `x` at the union of the members of `S` that `T` does not
    cover. For `S = unknown` that is `unknown` — no member of it is removed;
    for `S = string | int` with `T = string` that is `int`; for `S = string?`
    with `T = string` that is `null`. When no member remains — `S = T`, or
    `S = unknown` with `T = unknown` — the else branch never runs;
  - with `x is not T` the two branch types swap;
  - an assignment to `x` inside a branch ends the narrowing: from that point
    on, the branch checks with `x : S` again.
  A type that is not testable, or one that does not fit `S`, is `E0222`
  ([`errors.md`](../type-system/errors.md)). Narrowing belongs to the `if`
  condition alone: in every other `bool` position — `?:`, `while`, a
  `match` guard, the operands of `and` and `or`, under `!` — `is` is an
  ordinary test and `x`
  keeps type `S`, and a left operand that is not an identifier (a member
  access, a call) yields a `bool` but narrows nothing. The typing rules are
  in [`checking-rules.md`](../type-system/checking-rules.md).
- The condition is evaluated once; exactly one branch is evaluated.
- `if` is an expression. Its type is determined as follows:
  - with an `else` branch: both branches MUST have a common type
    ([`checking-rules.md`](../type-system/checking-rules.md)) and the `if` has
    that common type; branches with no common type are a compile-time error (`E0211`)
    ([`errors.md`](../type-system/errors.md));
  - without an `else` branch: the type is `unit`, and the value of the then
    branch, if any, is discarded. In particular, such an `if` cannot supply a
    value of any type other than `unit`.
- In statement position the rule quoted in the introduction applies.

```xulo
// expression position: both branches are `string`, so the type is `string`
let parity = if n % 2 == 0 { "even" } else { "odd" }
```

```xulo
fn describe(data: unknown) {
  if data is string {
    print(data)            // `data` is `string` here
  } else {
    print("not a string")  // `data` is `unknown` here: `unknown` minus
                           // `string` is not a type, so nothing is removed
  }
}

fn step(data: string | int) {
  if data is int {
    print(data + 1)        // `data` is `int` here
  } else {
    print(data)            // `data` is `string` here
  }
}

fn negated(data: unknown) {
  if data is not string {
    print("nope")          // `data` is `unknown` here
  } else {
    print(data)            // `data` is `string` here
  }
}
```

## `match` expressions

`match` tests a value against patterns and runs the body of the first arm
whose pattern matches and whose guard, when it has one, is `true`.

```xulo
let label = match x {
  0 => "zero"
  1 => "one"
  _ => "many"
}
```

- The scrutinee is evaluated exactly once, before any arm is tested, no matter
  how many arms are tried.
- Arms are tested in source order; the first arm whose pattern matches — and
  whose guard, when the arm has one, evaluates to `true` — wins and its body
  is evaluated. All later arms are skipped.
- An arm is `pattern [if condition] => expression` or
  `pattern [if condition] => { block }`; arms are separated by newlines or
  optional commas.
- A **guard** is an `if condition` written between the pattern and `=>`. It
  is evaluated only when that arm's pattern has matched, and at most once:
  when it is `false` the arm is skipped and testing continues with the next
  arm, so one `match` may evaluate several guards in source order. The
  condition MUST have type `bool` — there is no truthiness (`E0201`)
  ([`checking-rules.md`](../type-system/checking-rules.md)). The guard is
  evaluated with the bindings its arm's pattern made in scope, exactly as the
  body will be.
- `match` is an expression. In expression position all arm bodies MUST have a
  common type ([`checking-rules.md`](../type-system/checking-rules.md)); arms
  with no common type are a compile-time error (`E0211`)
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
| Type | `string s`, `unknown _`, `string \| int v` | a value whose type is the written type; binds the name for the body of the arm |
| Or | `p1 \| p2` | a value matched by any alternative; the alternatives bind the same names |
| Range | `a..<b`, `a...b` | numeric values inside the half-open or closed range |

- Enum variants are written with `::` in patterns and in expressions
  (`Shape::Circle(r)`), never with `.`.
- An **or-pattern** gives several alternatives separated by `|`
  (`0 | 1`, `true | false`, `Result::Ok(v) | Result::Err(v)`): the arm
  matches when any alternative matches, and an alternative MAY be a
  parenthesized pattern. Every alternative MUST bind the same names, each at
  the same type — both payloads of the `Result` example above bind `v` at one
  type — so that the guard and the body read one set of bindings;
  alternatives that disagree are a compile-time error (`E0304`)
  ([`errors.md`](../type-system/errors.md)).
- Sub-patterns nest arbitrarily: `Shape::Dot(Point(x, y))` matches a variant
  whose payload is a `Point` and then deconstructs that `Point`.
- Named payload fields are still matched positionally: `Shape::Rect(w, h)`
  matches a `Rect(width: int, height: int)` variant in declaration order.
- A payload or field position that is not bound MUST be written `_`:
  `Shape::Circle(_)` matches any radius without binding it.
- The operands of a range pattern MUST be numeric literals of the same numeric
  type as the scrutinee. `0...9` includes both endpoints, `0..<10` excludes the
  upper endpoint.
- **Type patterns** test the type of the scrutinee. The form is `Type binder`,
  where `binder` is an identifier or `_`: `string s` matches a value of type
  `string` and binds `s : string` for the arm's body, and `unknown _` accepts
  any `unknown` value without binding it. A type pattern is `match`'s type
  test; the same test outside `match` is the `is` operator (see
  [`if` expressions](#if-expressions)).
- The written type MUST be a **testable type**: a base type name (`int`,
  `string`, `bool`, `float`, `number`, the fixed-bit numerics, `null`,
  `unit`, `View`), a non-generic `struct` or `enum`, `unknown`, or an optional
  or union built from testable types (`string?`, `string | int`). A type
  argument list (`List<int>`, `Map<string, int>`) and a type parameter may
  never appear in a pattern, and any other type — a `List`, a `Map`, a
  function type, a generic instantiation — is not testable; a pattern naming
  one is a compile-time error (`E0303`)
  ([`../type-system/errors.md`](../type-system/errors.md)).
- The written type MUST be assignable to the scrutinee's type after alias
  expansion: `int i` fits a scrutinee of type `int` or `string | int`, but not
  one of type `string` (`E0303`). A designator that resolves to something that
  is not a type is `E0101`.
- **Tuples have no pattern form.** A tuple scrutinee is matched with a
  binding or `_`; to test its elements, read them with `p.0` or destructure
  with `let (a, b) = p` first, and `match` on those
  ([`../types/composite-types.md`](../types/composite-types.md)).
- Every pattern MUST be compatible with the type of the scrutinee; otherwise the
  `match` is a compile-time error (`E0303`).

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

fn tag(v: string | int): string {
  match v {
    string s => s
    int i => str(i)
  }
}

fn name(v: unknown): string {
  match v {
    string s => s
    null => "none"
    _ => "other"        // unknown is never exhausted by type patterns
  }
}
```

```xulo
fn bucket(n: int): string {
  match n {
    v if v < 0 => "negative"    // guard: the pattern alone matches everything
    0 | 1 => "zero or one"      // or-pattern: two alternatives, one arm
    v if v % 2 == 0 => "even"
    _ => "odd"
  }
}
// bucket(-3) => "negative"; bucket(1) => "zero or one";
// bucket(4) => "even"; bucket(7) => "odd"

match value {
  int x | string y => print(x)  // error: alternatives bind different names (E0304)
}
```

### Exhaustiveness and reachability

- An arm **covers** a type when its pattern matches every value of that type:
  a wildcard `_`, a binding `x`, or a type pattern of that exact type. The
  alternatives of an or-pattern cover together: `true | false` covers
  `bool`, and the members of a union may be split across alternatives. A
  **guarded** arm never covers — whatever its pattern, its guard may be
  false — so exhaustiveness counts only unguarded arms.
- A `match` MUST be exhaustive for the scrutinee's type. **`enum`**: every
  variant, by a variant pattern (possibly nested inside another pattern), or a
  covering arm. **`bool`**: `true` and `false`, or a covering arm. **A
  union**: every member — by a type pattern of that member, by its literal
  pattern when the member is a literal type, by `true` and `false` when the
  member is `bool`, or by a covering arm — and a covering arm is REQUIRED
  as soon as any member is not testable, because only a covering arm can
  match it; `string | int` is exhausted by `string s` and `int i`, and
  `"active" | "inactive"` by its two literals. **Any other type**: a covering
  arm, because literal and range patterns cannot cover a whole type and a type
  pattern can cover only a testable type exactly — in particular `List<int>`,
  a `Map`, and `unknown` need one, and `unknown` is exhausted only by an exact
  `unknown` pattern or a covering arm, never by enumerating concrete types.
- A `match` that fails these requirements is a compile-time error (`E0301`)
  ([`errors.md`](../type-system/errors.md)).
- An arm that can never be selected is an error (`E0302`). In particular, every
  arm after an unguarded wildcard or binding arm is unreachable, a literal
  that repeats an earlier unguarded literal arm is unreachable, and a type
  pattern whose type is already covered by earlier unguarded arms is
  unreachable. A guarded arm never makes a later arm unreachable — its guard
  may fail — and an or-pattern makes a later arm unreachable only when every
  value its alternatives match was already selected by earlier unguarded arms.

```xulo
fn describe(flag: bool): string {
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
  compile-time error (`E0213`) ([`errors.md`](../type-system/errors.md)).

| Iterable type | Loop binding | Iteration order |
|---------------|--------------|-----------------|
| `List<T>` | `T` | index order |
| `Array<T, N>` | `T` | index order |
| `Map<K, V>` | `K` | insertion order of keys |
| `Set<T>` | `T` | unspecified; programs MUST NOT depend on one |
| `Range<T>` | `T` | ascending from `start` |

- Iterating a `Map` visits its **keys**; the corresponding values are read
  with `m.get(k)` (see [`path-and-access.md`](path-and-access.md)). Iterating
  a `List`, `Array`, `Map`, `Set`, or `Range` is core syntax; the other
  operations on the collections are inherent members
  (see [`../types/composite-types.md`](../types/composite-types.md)).
- The loop variable is a fresh **immutable** binding for every iteration. It
  MAY shadow an outer binding of the same name, MUST NOT be assigned, and is
  not visible after the loop. To accumulate across iterations, use a `let mut`
  binding declared outside the loop. Because each iteration creates a distinct
  binding, a closure created during an iteration captures that iteration's
  binding (see [`closures.md`](closures.md)).
- The value of a `for` loop is `unit`.

```xulo
fn sum(xs: List<int>): int {
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
- Inside a list or map literal an element that starts with `...` is a spread
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
  `bool`; there is no truthiness
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
- Neither statement may appear in a `defer` operand, even inside a loop: the
  operand runs while control is already leaving a block, so there is no jump
  left to make, and the violation is `E0229`
  ([`../statements/return-and-block.md`](../statements/return-and-block.md)).
- `break` or `continue` outside a loop, or inside a function or closure body
  nested within a loop, is a compile-time error (`E0214`)
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

## Propagation and panic as control flow

Failure does not jump on its own: a caller moves it upward with the postfix
`?` operator, inspects a `Result` with `match`, and stops the program with
`panic(...)` when there is nothing to inspect. A loop contains none of them —
`?` inside a loop body performs the enclosing function's early return and
therefore leaves the loop, and a `panic` ends the program
([`../error-handling.md`](../error-handling.md)).

## Scoping rules

- Every block body — an `if` branch, a `match` arm, a loop body — opens a
  new block scope: a binding introduced inside is visible only inside that
  block; see [`names.md`](../names.md).
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
| `match e { ... }` | `e` once, then each arm's pattern in order, the guard of each arm whose pattern matched, then the body of the first arm selected | every later arm |
| `while c { B }` | `c` before each iteration, then `B` while `c` is `true` | `B` when `c` is `false` |
| `for x in it { B }` | `it` once, then one element binding and `B` per element | the whole body when the sequence is empty |
| `break` / `continue` | the jump itself | the rest of the current iteration; `break` also skips the rest of the loop |

`and` and `or` are word operators and short-circuit: the right operand may be
skipped entirely, so side effects placed in it do not necessarily happen.
