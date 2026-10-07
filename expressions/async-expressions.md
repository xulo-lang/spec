# Async Expressions

Asynchronous computation in Xulo is built on `async` functions and closures,
`await` expressions, and the built-in generic task type `Task<T>`. This chapter
defines their syntax and typing, the language-visible evaluation model, the
`Task` utilities, and where `await` may appear. Task scheduling — `spawn`,
cancellation, structured concurrency — is specified in
[`concurrency.md`](../concurrency.md), and error propagation in
[`error-handling.md`](../error-handling.md). The expression layer as a whole is
mapped in [`README.md`](README.md).

## `async` functions and closures

An `async` function is a function declaration written with the `async` keyword
before `fn` (after `pub` when present):

```xulo
async fn fetchUser(id: int): User {
  let raw = await request(`/users/${id}`)
  decodeUser(raw)
}

async fn notify() {
  print("done")
}
```

- The declared return type is written exactly as in a synchronous function,
  but it denotes the *declared* result: the value of the function name has the
  evaluated type `Task<T>` (see the table below).
- Inside an `async` body, the trailing expression and every `return` operand
  MUST have the declared type `T` — not `Task<T>`; wrapping the result into the
  task is automatic
  ([`checking-rules.md`](../type-system/checking-rules.md)).
- An `async` function without a declared return type has the evaluated type
  `fn(): Task<unit>` and its body produces `unit`.
- Calling an `async` function yields its `Task`; the result `T` is obtained
  with `await`.

| Written form | Value type |
|--------------|------------|
| `async fn f(): T { ... }` | `fn(): Task<T>` |
| `async fn f() { ... }` | `fn(): Task<unit>` |
| `async (x: A): R => ...` | `fn(A): Task<R>` |
| `async (x: A) => ...` | `fn(A): Task<R>` (result from the body) |
| `async fn(x: A): R { ... }` | `fn(A): Task<R>` |

An `async` closure is either form of closure ([`closures.md`](closures.md))
prefixed with `async`:

```xulo
async fn main() {
  let double = async (x: int): int => x * 2
  let t: Task<int> = double(21)
  print(await t)     // 42
}
```

## `await`

`await expr` waits for a task and produces its result.

- The operand MUST have type `Task<T>`; `await expr` then has type `T`. An
  `await` on a value that is not a `Task<T>` is a compile-time error
  ([`errors.md`](../type-system/errors.md)).
- The operand is evaluated exactly once, before the suspension it causes.
- `await` is legal only inside the body of an `async` function or an `async`
  closure. Elsewhere it is a compile-time error
  ([`errors.md`](../type-system/errors.md)).
- `await` is a prefix operator at the unary level of the precedence table
  ([`operators.md`](operators.md)): it binds tighter than any binary operator
  and looser than postfix operators such as calls, indexing, and member
  access.
- Inside an `async` body, `await` is an ordinary expression and may appear
  anywhere an expression may appear — including argument position.

```xulo
async fn greet(id: int): string {
  let user = await fetchUser(id)   // suspends here until the task settles
  "hello " + user.name
}

async fn sumPrices(a: int, b: int): int {
  let x = await priceOf(a)         // the second await does not begin
  let y = await priceOf(b)         //   until the first has completed
  x + y
}
```

```xulo
async fn demo(a: Task<int>, b: Task<int>) {
  let r1 = await cache.get("k")    // == await (cache.get("k"))
  let r2 = await a + await b       // == (await a) + (await b)
  print(await fetchCount())        // await in argument position
}
```

## Evaluation semantics

Calling an `async` function starts its body immediately. The body runs until it
either suspends at an `await` or completes:

- if the body completes without suspending, the call returns a `Task` that
  already holds the result;
- if the body suspends at an `await`, the call returns a `Task` representing
  the eventual result, and the remainder of the body runs when the awaited task
  settles.

`await` is the only suspension point visible in the language: no other
expression suspends an `async` body. Nothing further about scheduling is
language-visible — which task runs next, how many may run concurrently, and on
which execution resource are defined by the concurrency model
([`concurrency.md`](../concurrency.md)).

```xulo
async fn main() {
  let a = fetch(1)      // starts now, until its first suspension
  let b = fetch(2)      // starts now as well
  print(await a)        // resumes when `a` settles
  print(await b)
}
```

A `Task` may be awaited later rather than immediately; a task that is never
awaited leaves its result unused and is not thereby cancelled (see
[`concurrency.md`](../concurrency.md)).

## Task type and utilities

`Task<T>` is a built-in generic type (see
[`primitive-types.md`](../types/primitive-types.md)). It requires no import,
and the built-in `Task` namespace is accessed with `.`:

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Task.all` | `all<T>(tasks: list<Task<T>>): Task<list<T>>` | completes when every task has completed; results in argument order |
| `Task.race` | `race<T>(tasks: list<Task<T>>): Task<T>` | completes with the first task to complete |
| `Task.resolve` | `resolve<T>(value: T): Task<T>` | an already-completed task holding `value` |
| `Task.reject` | `reject<T>(err: Error): Task<T>` | an already-rejected task carrying `err` |

Generic type arguments are inferred at the call site. `Task.all` and
`Task.race` are commonly combined with `await`:

```xulo
async fn fetchAll(): list<User> {
  let a = fetchUser(1)
  let b = fetchUser(2)
  await Task.all([a, b])
}
```

## Errors in async code

- A `throw` executed inside an `async` body rejects the task; the remainder of
  the body does not run.
- The rejection is raised again at every `await` of that task, so `try`/`catch`
  written around an `await` handles it exactly like a synchronous error.
- An error raised before the first suspension rejects the task that the call
  returned in the same way.

```xulo
async fn logUser(id: int) {
  try {
    let user = await fetchUser(id)
    print(user.name)
  } catch e: NotFound {
    print("missing")
  }
}
```

Typed catch clauses, rethrowing, `finally`, and the error values carried by
rejected tasks are specified in [`error-handling.md`](../error-handling.md).

## Composition patterns

Awaiting an `async` call immediately after making it runs the work
sequentially. Calling several `async` functions first starts each of them at
its call, and awaiting their tasks afterwards lets the started work overlap:

```xulo
// sequential: each await completes before the next call is made
async fn sequential(): int {
  let a = await stepA()
  let b = await stepB()
  a + b
}

// concurrent: both tasks start at their calls, then both are awaited
async fn concurrent(): int {
  let ta = stepA()
  let tb = stepB()
  await ta + await tb
}
```

`Task.all` and `Task.race` combine several tasks into one. Starting work
independently of the current task — detached work, work on another execution
resource — is done with `spawn`, which is specified in
[`concurrency.md`](../concurrency.md).

## Where `await` is forbidden

`await` requires an `async` context. It is a compile-time error
([`errors.md`](../type-system/errors.md)) to write `await`:

- in the body of a function or closure that is not declared `async`, including
  a component body — a component is a function whose declared return type is
  `View`, not an `async` function;
- in an `@Effect` body, which is not declared `async`;
- at the top level of a module: module-level code is not an `async` context;
- outside any function or closure body;
- on a value whose type is not `Task<T>`.

Inside an `async` body no further restriction applies: `await` is an expression
and may appear in any expression position, including `f(await g())`,
conditions, initializers, and `for` iterables.

```xulo
async fn cached(id: int): User {
  if await isCached(id) {
    return await readCache(id)
  }
  await fetchUser(id)
}
```
