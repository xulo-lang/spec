# Error Handling

Xulo separates three situations. A **missing value** — an optional that holds
`null` — is not an error: it is ordinary data, and beyond it the language has
no `Option` type. A **failed operation** carries a reason: it returns the
built-in `Result<T, E>`, which a caller inspects with `match` or propagates
upward with the postfix `?`. An **unrecoverable condition** stops the program:
the program's own `panic(...)` call and the implementation's runtime failures
are the same event, and neither has a handler. This chapter specifies the
built-in type `Result`, the built-in type `Error` that serves as its
conventional payload, the `?` operator, `panic`, how results and cancellation
move through asynchronous code, and the table of runtime failures. Compile-time diagnostics are assigned in
[`type-system/errors.md`](type-system/errors.md); the ownership and storage
model is in [`memory-and-runtime.md`](memory-and-runtime.md); scopes,
cancellation, and services in [`concurrency.md`](concurrency.md).

## Absence and failure are orthogonal

- **`T?` says the value may be absent, and absence is not a failure.** A
  lookup that finds nothing is an outcome the caller usually expects; it is
  read with `??`, `?.`, a `null` test, or `?`, and it carries no reason and
  nothing to report.
- **`Result<T, E>` says the operation may fail, and `E` says why.** A refused
  connection, a malformed input: the caller may need the reason, so the
  reason travels as data.
- **When absence and failure are both possible, use `Result<T?, E>`:** `Err`
  is why the operation failed; `Ok(null)` is that it succeeded but found
  nothing.

```xulo
// Ok(user) found, Ok(null) absent, Err(reason) the lookup itself failed
fn find_user(id: int): Result<User?, LoadError> {
  if id < 0 { return Result::Err(LoadError::Denied("negative id")) }
  query(id)
}
```

- **Do not mix the two.** If the only outcome is "nothing was there", `T?`
  says so directly; reserving `Result` for operations that can fail for a
  reason keeps each `match` on its own subject.

## `Result<T, E>`

- **`Result` is a built-in generic type**, part of the prelude
  ([`builtins/prelude.md`](builtins/prelude.md)): no import introduces it, and
  the name is reserved, so a program cannot declare its own `Result`. It has
  exactly two variants, written with `::` like every other variant:

```xulo
Result::Ok(value)    // the operation succeeded
Result::Err(error)   // the operation failed
```

- `Result<T, E>` holds either a `T` or an `E`, and nothing else.
  `Result<int, string>` denotes one type in every module; the type is
  invariant in both arguments, exactly like an ordinary `enum`
  ([`types/type-relations.md`](types/type-relations.md)).
- A function that can fail returns a `Result` instead of raising anything:

```xulo
fn half(n: int): Result<int, string> {
  if n % 2 == 0 { Result::Ok(n / 2) } else { Result::Err("odd input") }
}

fn doubled_half(n: int): int {
  match half(n) {
    Result::Ok(v) => v * 2
    Result::Err(reason) => { print(reason); 0 }
  }
}
```

- **Failure is data, not control flow.** An `Err` is an ordinary return
  value: it is returned, bound, passed, and compared like any other value.
  Nothing jumps anywhere until the caller propagates it with `?`
  ([below](#the-operator)) or inspects it with `match`.
- **Exhaustiveness applies.** A `match` over a `Result` MUST cover both
  variants or have a wildcard or binding arm; an uncovered `Err` (or `Ok`) is
  `E0301`
  ([`expressions/control-flow.md`](expressions/control-flow.md)).
- **`Error` is the built-in base error type.** It is constructed by calling
  it with a message string — `Error("timed out")` — and carries one field,
  `message` of type `string`. `Result<T, Error>` is the conventional shape for
  system-level failures: file not found, connection refused, malformed input.
- **Domain failures are the program's own enums.** `Error` has no subtypes
  and Xulo has no inheritance, so a failure with structure is carried as an
  ordinary `enum` in the `E` slot — `Result<string, LoadError>` — and matched
  by name ([`types/enums.md`](types/enums.md)).

```xulo
enum LoadError { Missing(string), Denied(string) }

fn read_config(path: string): Result<string, LoadError> {
  if path == "" { return Result::Err(LoadError::Missing(path)) }
  load(path)
}
```

## The `?` operator

`?` is a postfix operator: `expr?` evaluates `expr` once and either yields its
**success value** or leaves the enclosing function through an **early
return** — the failure value, returned up one call.

| Operand type | Success value | On failure |
|--------------|---------------|------------|
| `T?` | the `T`, when the operand is not `null` | the enclosing function returns `null` |
| `Result<T, E>` | the `T` inside `Result::Ok` | the enclosing function returns the whole `Result::Err` value |

Rules:

- `e?` has the success type: `T` when `e : T?`, and `T` when
  `e : Result<T, E>`.
- An early return abandons the rest of the body at that point, exactly like a
  `return` statement written there
  ([`statements/return-and-block.md`](statements/return-and-block.md)).
- **The enclosing function MUST have a declared return type `R`, and the
  failure value MUST be assignable to `R`:** `null` for an optional operand
  (so `R` must be an optional type), and the whole `Result<T, E>` value for a
  `Result` operand (so — the type being invariant — `R` is that same
  `Result<T, E>`, or a union containing it). A `?` in a body with no declared
  return type, or one whose failure value does not fit `R`, is `E0217`.
- **The operand MUST be `T?` or `Result<T, E>`.** Any other operand — a plain
  `int`, a `Task<T>`, an `Error` — is `E0217`.
- Inside an `async` body, `await` and `?` combine with parentheses:
  `(await t)?`. Because `await` is a prefix operator and `?` a postfix one,
  `await t?` would try to propagate the *task* and is `E0217`.
- In statement position `e?` is an ordinary expression statement: when its
  success type is not `unit`, the statement-value rule applies (`E0218`)
  ([`statements/expression-statements.md`](statements/expression-statements.md)).

### Reading `?` beside the ternary

The ternary operator and the propagation operator share one spelling. When
`?` follows an expression it is read as the ternary operator whenever
`Expression ":"` follows it — whenever a complete ternary parses — and as the
postfix propagation operator otherwise. `f()?` propagates, `c ? a : b` is a
ternary, and `a? + b` propagates and then adds. The grammar states the same
rule ([`grammar.md`](grammar.md)); the two never compete in a well-formed
program.

### Examples

```xulo
fn quarter(n: int): Result<int, string> {
  let h = half(n)?     // Err("odd input") is returned from `quarter` as-is
  half(h)              // the trailing expression, itself a Result
}

fn fetch_name(id: int): string? {
  let u = find_user(id)?   // null propagates: `fetch_name` returns null
  u.name
}

fn still_wrong(n: int): int {
  half(n)?    // E0217: `Result::Err` is not assignable to `int`
}
```

`?` composes with optional chaining: `user?.profile?` propagates the missing
profile out of a function returning an optional, one step at a time.

## `panic`

`panic(message)` is an expression that stops the program where it is written.

- The operand MUST be a `string`; build a message with a template literal —
  `` panic(`bad count ${n}`) `` — or with the intrinsic `str`. The operand is
  evaluated exactly once, then the program stops. Nothing after the call
  runs, in any frame.
- **There is no handler.** `panic` cannot be caught, intercepted, or resumed;
  how the message is presented before the program stops is outside this
  specification.
- The expression may be given **any type the context requires**: a `panic`
  checks against every expected type, so a branch that cannot produce the
  branch type can stop instead. In statement position it is checked against
  `unit`, which always succeeds, so a bare `panic(...)` statement is
  well-formed.

```xulo
let label = if n > 0 { "positive" } else { panic("n must be positive") }

fn double_or_stop(n: int): int {
  if n < 0 { panic("negative input") }
  n * 2
}
```

- **Use `Result` for outcomes, `panic` for conditions.** A missing key, a
  rejected connection, an odd input: return `Err` — the caller may handle it.
  A broken invariant, a violated precondition, a state the program cannot
  describe: `panic` — handling it is meaningless.
- A `panic` inside an `async` body stops the whole program, not only the
  task: no task boundary contains it.
- A runtime failure ([below](#runtime-failures)) is the same event, raised by
  the implementation instead of by the program.

## Results in async code

An `async` body is ordinary control flow with one extension — suspension —
and its failure model is the same as a synchronous one:

- **Expected failures travel in the result.** `async fn f(): Result<T, E>`
  has the evaluated type `Task<Result<T, E>>`; `await` yields the `Result`,
  and `?` propagates from there — with parentheses, `(await t)?`.

```xulo
async fn fetch_user(id: int): Result<User, LoadError> {
  let raw = (await get_json(`/users/${id}`))?   // Err propagates out
  decode_user(raw)                              // Result<User, LoadError>
}
```

- **A body that ends in `Err` completes normally.** Its task settles with the
  `Result` value; an `Err` is data, not a failed task. There is no rejected
  task and no error to raise at an `await`.
- **A `panic` stops the program** from wherever in the body it runs.
- **Cancellation carries no error.** Awaiting a task that has settled as
  cancelled stops the awaiting body at that `await`: the await produces no
  value, and the awaiting task settles as cancelled in turn. The stop
  propagates outward through every enclosing `await` — until a scope absorbs
  its children's cancellation, or a cancelled `main` ends the program. There
  is nothing to catch, because no error value exists
  ([`concurrency.md`](concurrency.md)).

### `Task.all` and `Task.race`

The combinators
([`expressions/async-expressions.md`](expressions/async-expressions.md))
combine results and cancellation the way the operations they describe settle:

- `Task.all` waits until every task of the list has settled, then completes
  with the results in argument order; if any of them settled as cancelled, the
  combined task settles as cancelled.
- `Task.race` settles as soon as the first task of the list settles: with
  that task's result when it completed, as cancelled when it was cancelled.
- Neither combinator cancels the tasks it combines: the others run on and
  their results are discarded when they are not the ones selected.

## Runtime failures

A runtime failure stops the program where it occurs: it is a panic raised by
the implementation, and — like the program's own `panic(...)` — it has no
handler. The table below is identical to the one in
[`memory-and-runtime.md`](memory-and-runtime.md), which is normative for the
operations themselves.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index out of bounds | reading or writing `xs[i]` where `i` is negative or at or past the list's length | [`types/composite-types.md`](types/composite-types.md) |
| Absent map key | reading `m[k]` for a key with no binding | [`expressions/path-and-access.md`](expressions/path-and-access.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |

[`concurrency.md`](concurrency.md) defines two further runtime failures of its
own — misuse of `lock` and exhaustion of the task limit while spawning.

A `Result`, by contrast, is data: it is returned, bound, propagated with `?`,
and matched. Nothing a program can recover from ever stops the program, and
nothing that stops the program can be recovered from.
