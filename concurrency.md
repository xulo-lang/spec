# Concurrency

This chapter specifies Xulo's concurrency model: tasks and the `spawn`
expression, the executors that run spawned blocks, capture rules for spawned
work, scopes and structured concurrency, cancellation, shared state guarded by
`lock`, and message-passing services. The evaluation and ownership rules that
concurrency builds on are in
[`memory-and-runtime.md`](memory-and-runtime.md); `async`, `await`, and the
`Task` utilities are in
[`expressions/async-expressions.md`](expressions/async-expressions.md);
results, propagation, and `panic` in
[`error-handling.md`](error-handling.md).

## Tasks

A **task** is a unit of asynchronous computation with type `Task<T>`
([`README.md`](README.md)). Calling an `async` function or closure starts its
body in the current task's context; `await` is the only suspension point the
language exposes. `spawn` starts work whose body is not part of any call: the
spawning expression yields a `Task` immediately, the body runs as its own
task, and the task settles when the body completes or is cancelled.

- Starting work with an ordinary `async` call ties the result to that call;
  starting it with `spawn` detaches the body from the call chain while
  keeping it attached to the block that spawned it
  ([Scopes and structured concurrency](#scopes-and-structured-concurrency)).
- A task runs its body to completion or until it suspends at an `await`; no
  other expression suspends a body.
- Tasks are values: a `Task<T>` may be stored, passed, returned, and awaited
  later. A task that is never awaited leaves its result unused and is not
  thereby cancelled.

## The `spawn` expression

`spawn` has five forms — four that run a block, and one that starts a
service:

```text
spawn async { Block }
spawn.thread async { Block }
spawn.process async { Block }
spawn.on(Pool) async { Block }
spawn.process Type { FieldInit , … }
```

| Form | Runs the block |
|------|----------------|
| `spawn async { … }` | on the default task scheduler |
| `spawn.thread async { … }` | on an operating-system thread |
| `spawn.process async { … }` | in a separate process |
| `spawn.on(pool) async { … }` where `pool` has type `fn(fn(): unit): unit` | where `pool` decides |
| `spawn.process Type { … }` | as a service of type `Type` ([Services](#services)) |

- The `async` keyword is required in the block forms, and the block is an
  `async` context: `await` is legal inside it wherever it is legal inside an
  `async` function body
  ([`expressions/async-expressions.md`](expressions/async-expressions.md)).
- The first four forms are expressions of type `Task<T>`, where `T` is the
  type of the block. The service form evaluates to a service handle.
- The dispatch mode selects the executor that runs the block's segments
  ([Executors](#executors)). Capture, scope, and cancellation rules are
  identical in every mode.
- `spawn` is an ordinary expression: it may appear in a synchronous or an
  `async` body, in any expression position its type permits.
- A `spawn` expression is not `unit`, so it may not stand alone as an
  expression statement ([`statements/README.md`](statements/README.md)).
  Bind it, `await` it, or pass it on.

```xulo
async fn main() {
  let first = spawn async { weight(3) }
  let second = spawn.thread async { weight(4) }
  print(await first + await second)   // 7
}
```

### Executors

The dispatch mode selects the **executor** that runs the block: the default
task scheduler, an operating-system thread, a separate process, or a
caller-supplied pool. The service form does not produce a task; its process
is governed by [Services](#services). A task's execution is a sequence of
**segments**: the body runs until it either completes or suspends at an
`await`, and each segment is handed to the task's executor when the task
becomes ready to run — once when the `spawn` expression is evaluated, and once
more after each `await` in the block completes.

- A segment runs from the task's current position to its next suspension or
  to settlement. The executor MAY begin the segment before the call that
  hands it over returns; the `spawn` expression itself yields the `Task`
  without waiting for any segment to finish.
- The result of the block is carried by the `Task` itself, never through the
  executor: the `fn(): unit` a pool receives runs one segment and returns
  `unit`, and the value the block produces is stored in the task when it
  settles.
- For `spawn.on(pool)`, each ready transition calls `pool` with one segment
  thunk. The pool MAY run it immediately, queue it, or hand it to a thread.
  A pool that never invokes a thunk leaves the corresponding task unsettled,
  so a scope draining that task waits indefinitely. The runtime never hands
  one task two thunks at once.
- For `spawn.thread`, each segment runs on an operating-system thread, which
  MAY be released when the segment ends and MAY differ from segment to
  segment. No language rule depends on which thread a segment runs on.
- For `spawn.process async { … }`, segments run on the new process's own
  scheduler, and the block's result is carried back to the `Task` in the
  spawning process when the block settles.
- A task cancelled while its next segment waits in an executor settles as
  cancelled when that segment runs; the body does not resume.

## Capture into a `spawn` block

A `spawn` block is not a closure. It does not see the bindings of the scope
that surrounds it; it must be given what it uses.

- Always visible inside a block, without capture: declarations (`fn`,
  `struct`, `enum`, `trait`, `impl`, `type`), `const` bindings, and the
  prelude's intrinsics and names.
- Every other enclosing binding — a parameter, a local `let` or `let mut`, a
  module-level `let` or `let mut` — MUST be captured before the block's body
  may use it. A capture is written as a `let` initializer in the block:
  `let local = move config` or `let local = copy config`. Mentioning an
  uncaptured enclosing binding anywhere else in the block is a compile-time
  error (`E0408`).
- `move` transfers ownership: the capture is evaluated when the `spawn`
  expression is evaluated, and the enclosing binding is moved out
  ([`memory-and-runtime.md`](memory-and-runtime.md)); using it after the
  `spawn` is an error (`E0403`).
- `copy` takes a snapshot at the same moment and leaves the enclosing
  binding usable.
- A `shared` binding needs no capture form: the block captures the shared
  region itself, and its contents are reachable only under `lock`
  ([Shared state and `lock`](#shared-state-and-lock)).

Capture is decided when the `spawn` expression is evaluated, before the body
runs, so what a block receives never changes while it executes:

```xulo
fn upload_files(): Task<unit> {
  let config = load_config()
  let data = gather()
  let job = spawn.thread async {
    let local_config = move config   // captured when the spawn runs
    let local_data = copy data       // a snapshot; `data` stays usable
    transmit(local_data, local_config)
  }
  print(data)                        // legal: `data` was copied, not moved
  // print(config)                   // error: `config` has been moved (E0403)
  job                                // returned: this task outlives the scope
}
```

## Scopes and structured concurrency

Every block is a **scope** for the tasks started inside it — by `spawn` or
by an `async` call — including a function or closure body and the body of
`main`. When a block finishes, it does not simply stop: it settles its tasks
first, then runs its `defer`s.

1. The block's value, if any, is determined.
2. Each task started in the block that has not settled is cancelled.
3. The block waits until every one of those tasks has settled.
4. Each settled child contributes nothing further: a child that settled as
   cancelled is absorbed, and a result no longer reachable from the block's
   value is discarded.
5. Each `defer` registered in the block runs, in reverse order of
   registration ([`statements/return-and-block.md`](statements/return-and-block.md)).
6. The block's value is produced.

A task **reachable from the block's value** is exempt from step 2: it is not
cancelled, and its ownership passes to the enclosing scope together with the
value, where the same rules apply. This is how a function returns a task it
started — whether by `spawn` or by an `async` call.

Three consequences follow:

- **Parent cancellation propagates downward.** Cancelling a task cancels the
  scope inside its body, which cancels that scope's tasks in turn.
- **The program ends when `main` returns.** `main` may be declared `async`
  ([`modules/source-files.md`](modules/source-files.md)); when it is, the
  program ends when its task settles — completed or cancelled — by which time
  its body's scope has already drained. Work that must finish before exit is
  awaited inside the scope that owns it.
- **Services outlive the block that started them.** A service is not a task
  ([Services](#services)): a block's completion neither cancels nor stops it,
  and its lifetime is governed by its handle
  ([Service lifetime and `stop`](#service-lifetime-and-stop)).

## Cancellation

Cancellation is cooperative and requested only by the language — when a scope
exits ([above](#scopes-and-structured-concurrency)) — and propagates to a
cancelled task's own scope.

- A cancelled task stops at its next `await`: the pending `await` produces no
  value and the task settles as cancelled. A task that is running continues
  until it suspends.
- Before it settles, a cancelled task runs the defers registered in its live
  blocks — innermost block first, each in reverse order of registration. An
  `await` inside such a defer ends that run where it stands, like any other
  `await` of a cancelled task: the remainder of that defer and every defer
  still waiting are abandoned, and the task settles
  ([`statements/return-and-block.md`](statements/return-and-block.md)).
- Cancellation carries no error value: no expression observes it, nothing
  raises it, and there is nothing to catch. The body runs as written between
  the cancellation request and its next `await`.
- Cancellation is final: a cancelled task settles as cancelled whatever its
  body does afterwards, and a value the body produces after settling is
  discarded.
- `await` of a cancelled task stops the awaiting body at that `await`, which
  produces no value, and the awaiting task settles as cancelled in turn — the
  propagation continues outward through every `await` until a scope absorbs
  it or a cancelled `main` ends the program
  ([`error-handling.md`](error-handling.md)).
- A scope that cancels its own children absorbs their cancellation: they
  settle as cancelled and the scope continues normally.
- Cancellation cannot interrupt code between suspension points: a body that
  never awaits again runs to completion even though it has been cancelled.

## Shared state and `lock`

Concurrent tasks mutate one value only through a **shared region** guarded by
a lock. `shared` is part of a binding form, exactly as `mut` is: it appears
in a `let` and in a parameter declaration, and it adds no type — a shared
binding of `AppState` has type `AppState`.

```text
let state = shared Expr          // a shared binding
fn f(state: shared T, …)         // a shared parameter
lock state { Block }             // a critical section
```

- The contents of a shared binding — every field and subscript reached
  through it — MAY be read or written only while the current task holds the
  lock on that binding, that is, lexically inside `lock state { … }` on that
  very binding, or inside a function called from such a section that receives
  the region as a `shared` parameter. Any other access is a compile-time
  error (`E0406`).
- Inside `lock state { … }` the contents reached through `state` are mutable
  places, whether or not the binding is `let mut`; assignment through them
  follows the ordinary place rules.
- The operand of `lock` MUST be a shared binding (`E0407`), as must an
  argument passed to a `shared` parameter.
- `lock` is an expression: its value is the value of the block, so a section
  both mutates and produces. The lock is released when the block finishes —
  normally, or by `return`.
- A `lock` body contains no suspension point: `await` inside `lock` is a
  compile-time error (`E0410`). A critical section therefore runs from entry
  to exit without yielding, and cancellation — which takes effect at a task's
  next `await` — cannot interrupt one.
- Acquiring a lock never suspends the acquiring task. On a runtime that runs
  several tasks on one thread, a critical section's non-suspending nature
  means two tasks on that thread can never contend for one binding's lock. On
  a runtime that runs tasks in parallel, the executing thread waits — blocked
  or spinning — until the section in progress finishes; because that section
  contains no `await`, the wait is bounded by its duration. Waiting this way
  is not a suspension point, so `await` remains the only one
  ([`expressions/async-expressions.md`](expressions/async-expressions.md)).
- Locks are not reentrant: acquiring a lock the current task already holds is
  a runtime failure defined in [Runtime failures](#runtime-failures).

```xulo
struct AppState { total_online: int, names: List<string> }

fn join(state: shared AppState, name: string) {
  lock state {
    state.total_online = state.total_online + 1
    state.names = state.names + [name]
  }
}

fn report(state: shared AppState): int {
  lock state { state.total_online }
}
```

The no-suspension rule draws a line between where a value is fetched and where
it is installed; the await belongs outside the section:

```xulo
async fn refresh_bad(state: shared AppState) {
  lock state {
    state.total_online = await fetch_count()   // error[E0410]
  }
}

async fn refresh_ok(state: shared AppState) {
  let fresh = await fetch_count()              // ok: the await is outside the lock
  lock state { state.total_online = fresh }
}
```

Sections should be short, and a task that needs several locks SHOULD acquire
them in a consistent order; tasks that wait on one another's locks make no
further progress.

## Memory visibility and atomics

Xulo exposes no atomic types, no fences, and no memory-order parameters.
Nothing of the kind is needed: the checker confines every access to shared
storage inside a `lock` on that binding (`E0406`, `E0407`), spawn blocks
capture by `move` or `copy` rather than by reference, and service messages
are deep-copied in transit — so a shared region guarded by its lock is the
only storage two tasks can both reach, and every access to it is already
ordered by construction.

An implementation MAY use atomics, fences, operating-system locks, or a
single thread internally; none of that is observable. What it MUST preserve,
for every conforming program, is the ordering the language does state:

- **Within a task**, operations are sequenced: evaluation order is the
  left-to-right, source-order discipline fixed in
  [`memory-and-runtime.md`](memory-and-runtime.md).
- **Across a lock**, critical sections on one shared binding are ordered:
  the writes performed inside a `lock state { … }` are visible to every
  critical section on `state` that begins after it finishes, whichever task
  performs it. Release — normal exit or `return` — publishes; the next
  acquire observes.
- **By value**, everywhere else: the result of `await`, a settled task's
  result, and a message are values carried between tasks
  ([`memory-and-runtime.md`](memory-and-runtime.md)), not shared storage.

Data-race freedom is therefore a consequence of the static rules, not an
obligation the programmer carries: two tasks cannot touch one cell of shared
storage unless both do so inside `lock` on its binding, and outside shared
storage no two tasks hold the same storage at all. What remains free —
which ready task runs next, how many run concurrently — is the subject of
[Scheduling and progress](#scheduling-and-progress).

## Services

A **service** is an actor: a struct with an `on_message` method, running in
its own process, reached only by sending messages.

```xulo
struct Tally { total: int }

impl Tally {
  fn on_message(mut self, cmd: int) {
    self.total = self.total + cmd
    reply(self.total)
  }
}

async fn main() {
  let worker = spawn.process Tally { total: 0 }
  let sum = await worker.send(41)
  print(sum)                        // 41
  worker.stop()                     // the service ends; nothing else is queued
}
```

- `spawn.process Type { field: expr, … }` starts a service of the struct type
  `Type`. The field list initializes the service's state under the rules of
  struct construction ([`types/composite-types.md`](types/composite-types.md));
  it is evaluated left-to-right, and the constructed value is deep-copied into
  the new service, so the service never shares storage with its spawner.
- `Type` MUST declare `on_message` in an `impl` block; a service type without
  one is a compile-time error (`E0409`). The method takes a receiver (`self`
  or `mut self`) and exactly one message parameter of some type `M`; it MAY be
  declared `async`.
- The spawn evaluates to a **service handle**: a value of the structural type
  `{ send: fn(M): Task<R>, stop: fn(): unit }`, whose `send` member takes a
  value of `M` and whose `stop` member ends the service
  ([Service lifetime and `stop`](#service-lifetime-and-stop)).
- `handle.send(m)` copies the message value into a message, queues it for the
  service, and returns a task of type `Task<R>` immediately. The message is a
  value ([`memory-and-runtime.md`](memory-and-runtime.md)): it is deep-copied
  in transit and the sender's value is unaffected. When the operand is
  `move x`, ownership of the message instead transfers to the service: no copy
  is performed, and any later use of `x` is a compile-time error (`E0403`),
  exactly as after any other `move`
  ([`memory-and-runtime.md`](memory-and-runtime.md)). Because the copy
  completes before `send` returns, the argument's borrow — if it had one —
  ends with the call and does not span the returned task
  ([`memory-and-runtime.md`](memory-and-runtime.md), Span end).
- The queue of messages waiting for a service is unbounded: `send` never
  waits for capacity and never fails because the queue is full. A client that
  needs backpressure limits how many sends it leaves unanswered.
- The returned `Task<R>` MAY be discarded: `handle.send(m)` in statement
  position is fire-and-forget — the message was queued when the call
  evaluated, so only the reply goes unread, and the block settles the task on
  exit ([`statements/expression-statements.md`](statements/expression-statements.md)).
- A service handles one message at a time, in arrival order: `on_message` runs
  to completion — through every `await`, if it is `async` — before the next
  message begins.
- `reply(e)` may be called only inside `on_message`. The reply type `R` is the
  common type of the `reply` operands in the method (`E0211` if they have
  none), or `unit` when the method never calls `reply`. The send's task
  settles when `on_message` returns, carrying the operand of the `reply` the
  executed path performed — or, when that path performed none although the
  method contains `reply` calls, settling as cancelled.
- Cancelling a send task does not withdraw a message already queued.

### Service lifetime and `stop`

A service is not bound to the block that started it. It runs until it is
stopped through its handle or until the program ends:

- Completing the block that evaluated the `spawn.process` expression does not
  stop the service. The handle may be stored, passed, and used after that
  block has finished — including from a later iteration of a loop whose body
  started the service.
- `handle.stop()` requests that the service stop. Stopping is cooperative: it
  follows the same rule as a cancelled task
  ([Cancellation](#cancellation)) — the service finishes its current
  synchronous run and stops at its next `await`. A body that never awaits
  again runs to completion, and the service has stopped when `on_message`
  returns.
- When a service stops, every message still queued for it is discarded, and
  each discarded message's send task settles as cancelled. A `reply` performed
  by the path that was executing is unaffected.
- A `send` performed after the service has stopped is discarded immediately
  and its task settles as cancelled, exactly like a message still queued when
  the service stopped.
- `stop` is idempotent: calling it on a service that has already stopped — or
  from several holders of the handle — has no further effect.
- When the program ends, every service that has not stopped is stopped and its
  queued messages are discarded.

Messages are the only channel: there is no shared state between a service and
its clients, so a service never needs `shared` or `lock`.

## Scheduling and progress

Nothing about scheduling is language-visible: which ready task runs next, how
many run concurrently, and how long a suspension lasts are outside the
specification, and programs MUST NOT depend on them. Which executor runs a
mode's segments is fixed ([Executors](#executors)); which thread or process
instance within that executor is not. What is fixed — evaluation order, scope
completion, message arrival order, the order of `Task::all` results — is listed
under [`memory-and-runtime.md`](memory-and-runtime.md).

## Runtime failures

This chapter defines the runtime failures peculiar to concurrency; the
common table of runtime failures is in
[`memory-and-runtime.md`](memory-and-runtime.md), and each of them stops the
program as a panic ([`error-handling.md`](error-handling.md)).

| Condition | When it occurs |
|-----------|----------------|
| Lock misuse | a task acquires a `lock` it already holds (locks are not reentrant) |
| Task exhaustion | the runtime's limit on live tasks is exceeded while spawning |

## The `Task` utilities

`Task::all`, `Task::race`, `Task::resolve`, and `Task::timeout` combine and produce
tasks; their signatures are given identically in
[`expressions/async-expressions.md`](expressions/async-expressions.md) and
[`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md). How
results and cancellation combine through `Task::all`, `Task::race`, and
`Task::timeout` is specified in [`error-handling.md`](error-handling.md).
