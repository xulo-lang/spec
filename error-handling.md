# Error Handling

Xulo distinguishes two runtime mechanisms. A **thrown value** is data the
program raises deliberately and that a handler may catch: `throw` sends it to
the nearest `try`. A **runtime failure** — an out-of-range subscript, a
division by zero — is not a value and has no handler. This chapter specifies
`throw`, `try`/`catch`/`finally`, the built-in type `Error`, the `Result`
idiom, how errors move through asynchronous code, and the table of runtime
failures. Compile-time diagnostics are assigned in
[`type-system/errors.md`](type-system/errors.md); the ownership and storage
model is in [`memory-and-runtime.md`](memory-and-runtime.md); scopes,
cancellation, and services in [`concurrency.md`](concurrency.md).

## Thrown values

- **Any value may be thrown.** `throw` takes an operand of any type, and the
  thrown value is what a handler sees.
- By convention a thrown value is a domain `enum`
  ([`types/enums.md`](types/enums.md)) or an `Error`, so that `catch` clauses
  can name the failure precisely. Other types are legal but rarely useful.
- **`Error` is the built-in base error type.** It is constructed by calling
  it with a message string — `Error("timed out")` — and carries one field,
  `message` of type `string`. `Error` is the type carried by rejected tasks
  (`Task.reject`) and the conventional payload of a thrown system-level
  failure.
- `Error` has no subtypes and Xulo has no inheritance: a domain `enum` thrown
  as a value is not an `Error`, and a `catch e: Error` clause matches only
  thrown `Error` values. A clause that must match every thrown value is the
  untyped `catch e` or `catch _` ([below](#try-and-catch)).

Thrown values and runtime failures are different in kind: failures are listed
in [Runtime failures](#runtime-failures) and cannot be caught.

## `throw`

`throw` is a statement: it abandons the current control path and transfers
control to the nearest enclosing handler — the innermost `catch` clause of an
enclosing `try`, searching outward through function, closure, and block
boundaries ([`statements/README.md`](statements/README.md)).

```xulo
enum LoadError { Missing(string) }

fn read_config(path: string): string {
  if path == "" { throw LoadError::Missing(path) }
  load(path)
}
```

- The operand is evaluated exactly once, before control transfers; the
  resulting value is the value the handler receives.
- A `throw` inside a loop leaves the loop: a loop catches nothing by itself
  ([`expressions/control-flow.md`](expressions/control-flow.md)).
- `throw` inside a `catch` or `finally` clause raises the new value to the
  next enclosing handler, after `finally` blocks of the construct it leaves
  have run.
- When no handler exists, the thrown value reaches the program boundary and
  the program stops with the runtime failure *uncaught `throw`*.

## `try` and `catch`

```text
try { Block }
  [ catch e { Block } | catch e: Type { Block } | catch _ { Block } ] …
  [ finally { Block } ]
```

A `try` construct has at least one `catch` or `finally` clause; a construct
with neither is a compile-time error (`E0217`). Clauses are written in order,
each `catch` before `finally`, and `finally` last.

### The three clause forms

| Form | Effect |
|------|--------|
| `catch e: T` | handles a thrown value whose type is `T`, binding it to `e` |
| `catch e` | handles every thrown value, binding it to `e` |
| `catch _` | handles every thrown value, discarding it |

- **Selection is first match wins.** The clauses are tested in the order
  written, and the first clause whose test succeeds handles the value; no
  later clause is considered. A clause that can never be reached because an
  earlier clause already matches every value is reported as unreachable code
  (`W0103`).
- **The type test uses subtyping, without coercion.** A `catch e: T` clause
  succeeds when the dynamic type of the thrown value is a subtype of `T`
  ([`types/type-relations.md`](types/type-relations.md)). The value bound to
  `e` then has type `T`.
- **An untyped `catch e` binds a value of undetermined type.** It matches
  everything, but reading `e` anywhere its type is needed is a compile-time
  error (`E0216`); write `catch e: T` to inspect the value, or `catch _` to
  ignore it.
- The binding of `catch e` or `catch e: T` is scoped to that clause's block
  alone, exactly like `_` in a pattern
  ([`names.md`](names.md)); clauses may reuse the same name.

### The value of `try`

`try` is an expression. Its type is the common type of the `try` block and of
every `catch` block ([`type-system/checking-rules.md`](type-system/checking-rules.md));
a clause block that ends by abandoning the path — a `return`, `break`,
`continue`, or `throw` — completes with no value and imposes no constraint.
`finally` contributes nothing to the type. In statement position `try` is an
expression statement covered by the statement-value rule's exception: its
value — like that of `if` and `match` — is discarded, whatever its type
([`statements/expression-statements.md`](statements/expression-statements.md)).

```xulo
fn safe_half(n: int): int {
  try {
    if n % 2 != 0 { throw LoadError::Missing("odd input") }
    n / 2
  } catch e: LoadError {
    0
  } finally {
    print("checked")
  }
}
```

## `finally`

A `finally` block runs on every path out of its `try` construct, in all of
these cases:

1. the `try` block completes normally — the value of `try` is computed after
   `finally` runs;
2. a `catch` clause handles a thrown value — the handler's block completes,
   then `finally` runs;
3. a thrown value is not handled by any clause of this construct — `finally`
   runs before the value continues to the next enclosing handler;
4. control leaves early — a `return`, `break`, or `continue` inside the `try`
   or a `catch` block takes effect only after `finally` runs.

`finally` neither produces nor consumes the construct's value or error: it
cannot change what `try` evaluates to, and it cannot replace a propagating
value. To keep that guarantee, `return`, `break`, `continue`, and `throw` MUST
NOT appear in a `finally` block; a violation is a compile-time error
(`E0217`).

```xulo
fn read_or_default(path: string): string {
  try {
    let text = read(path)
    text
  } catch e: LoadError {
    "default"
  } finally {
    print("done")
  }
}
```

## The `Result` idiom

Some failures are ordinary outcomes, not exceptions: a missing key, an odd
input, a queue that is empty. For these the program declares a result enum
and returns it, so that handling is exhaustive and checked by `match`
([`types/enums.md`](types/enums.md)):

```xulo
enum Result<T, E> { Ok(T), Err(E) }

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

- `Result` is an ordinary declaration, not a built-in: the program writes it
  once with exactly these variants — `Result::Ok(T)` and `Result::Err(E)` —
  and imports or re-exports it like any other type.
- There is no propagation operator: a caller unwraps a `Result` with `match`,
  in the same way it handles an error with `try`.
- The two mechanisms compose. A function may return a `Result` for expected
  outcomes and still `throw` for programming errors; `catch` and `match`
  remain independent.

## Errors in async code

An `async` body is ordinary control flow with one extension — suspension —
and its errors follow:

- A `throw` executed inside an `async` body rejects the task; the remainder
  of the body does not run, and the task carries the thrown value
  ([`expressions/async-expressions.md`](expressions/async-expressions.md)).
- A rejection is raised again at every `await` of that task, so `try` around
  an `await` handles it exactly like a synchronous throw.
- A rejection that no `await` observes is raised when the scope that owns the
  task completes: a task spawned in a block settles when the block exits, and
  an uncaught error it carries is raised there and propagates outward through
  enclosing scopes ([`concurrency.md`](concurrency.md)). A rejection reaching
  the program boundary stops the program (uncaught `throw`).
- The cancellation of a task is raised at `await` as an ordinary thrown
  value of type `Error`; a body or an awaiting task may catch it to run
  cleanup, although cancellation itself cannot be undone
  ([`concurrency.md`](concurrency.md)).

```xulo
async fn main() {
  try {
    let text = await load("")
    print(text)
  } catch e: LoadError {
    print("missing")
  } finally {
    print("done")
  }
}
```

### `Task.all` and `Task.race`

The combinators
([`expressions/async-expressions.md`](expressions/async-expressions.md))
propagate errors like the operations they describe:

- `Task.all` waits until every task of the list has settled, then rejects if
  any of them rejected, with the error of the first rejected task **in
  argument order**. A task that settled as cancelled counts as rejected with
  the cancellation error.
- `Task.race` settles as soon as the first task of the list settles; if that
  task rejected, the race rejects with its error, and if it settled as
  cancelled, the race is cancelled.
- Neither combinator cancels the tasks it combines: the others run on and
  their results are discarded when they are not the ones selected.

## Runtime failures

A runtime failure stops the program where it occurs; there is no handler for
it, and `try` never intercepts one. The table below is identical to the one
in [`memory-and-runtime.md`](memory-and-runtime.md), which is normative for
the operations themselves.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index out of bounds | reading or writing `xs[i]` where `i` is negative or at or past the list's length | [`types/composite-types.md`](types/composite-types.md) |
| Absent map key | reading `m[k]` for a key with no binding | [`expressions/path-and-access.md`](expressions/path-and-access.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |
| Uncaught `throw` | a thrown value escapes every enclosing handler, including at the program boundary | [`error-handling.md`](error-handling.md) |

[`concurrency.md`](concurrency.md) defines two further runtime failures of its
own — misuse of `lock` and exhaustion of the task limit while spawning.

Thrown values, by contrast, are values: they travel through `throw`, are bound
by `catch` clauses, are carried by rejected tasks, and can be inspected,
rethrown, and matched. Only their final escape — with no handler left —
becomes the runtime failure *uncaught `throw`*.
