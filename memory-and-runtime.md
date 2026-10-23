# Memory and Runtime

This chapter defines how Xulo values occupy memory and how programs use them:
the value and reference semantics of every type, the ownership forms `move` and
`copy`, parameter borrowing, mutability, informal lifetimes, storage
reclamation, and the runtime failures that stop a program. Compile-time
diagnostics are assigned in [`type-system/errors.md`](type-system/errors.md);
the concurrency rules that build on this model — `spawn`, `shared`, `lock`, and
cancellation — are specified in [`concurrency.md`](concurrency.md); the error
model — `Result`, `?`, and `panic` — in
[`error-handling.md`](error-handling.md).

## Evaluation model

Evaluation produces values; a value occupies storage until nothing can reach
it (see [Garbage collection](#garbage-collection)). The evaluation order rules
of [`expressions/README.md`](expressions/README.md) are normative and hold
everywhere: operands, call arguments, and literal elements are evaluated
left-to-right, statements in a block run in source order
([`statements/return-and-block.md`](statements/return-and-block.md)), and
module-level initializers run in declaration order before `main`
([`modules/source-files.md`](modules/source-files.md)).

Two notions are used throughout this chapter:

- A **place** is an expression that designates storage: a path
  (`count`), a field through a place (`p.x`), or a subscript through a place
  (`xs[0]`). Reading a place yields the value it holds; writing to a place
  replaces that value and requires the place to be mutable
  ([`statements/let-and-assignment.md`](statements/let-and-assignment.md)).
- A **value expression** designates a value without naming storage:
  a literal, a computation, a call. Most expressions are value expressions;
  `move` and `copy` below constrain where they may appear.

A binding *denotes* a value: `let x = e` evaluates `e` once and associates the
resulting value with `x`. What that association means — a fresh independent
value or a shared reference — is the subject of the next section.

## Value semantics and reference semantics

Every type has one of two categories. The category decides what a binding
holds and what an assignment does.

| Category | Types | A binding denotes | `let x = e` and assignment |
|----------|-------|-------------------|----------------------------|
| **Value** | `bool`, `string`, `int`, `float`, `number`, fixed-bit numerics, `null`, `unit`, `struct`, `enum`, `Range<T>`, tuples | the value itself | a fresh, independent copy of `e`'s value; a `struct`, `enum`, or tuple copy is shallow — each field or element is copied by its own category |
| **Reference** | `List<T>`, `Map<K, V>`, `Set<T>` | a reference to one shared value | the same value, shared: `x` and `e` then denote the one underlying value |

Four further values carry **identity** rather than content — function values
(closures), `Task<T>`, `View`, and `Error` — and behave as references to a
single, distinct thing: two such values are the same only when they are the
same value ([Equality and identity](#equality-and-identity)).

Consequences that follow directly from the table:

- Value-semantic values cannot be aliased. A write through one binding is
  never observable through another, because the bindings hold different
  copies.
- Reference values are shared. A write through any mutable place naming the
  value is visible through every other binding that names it. Independence is
  obtained explicitly with `copy` ([Ownership with `copy`](#ownership-with-copy)).
- `move` is what severs a binding from a value of either category
  ([Ownership with `move`](#ownership-with-move)).

```xulo
struct Point { x: int, y: int }

let a = Point(x: 1, y: 2)
let mut b = a        // b holds an independent copy of a's value
b.x = 9
print(a.x)           // 1: the write through b never reaches a

let xs = [1, 2, 3]
let ys = xs          // ys and xs denote the same list
let mut zs = ys
zs[0] = 9
print(xs[0])         // 9: the write through zs is visible through xs
```

Arguments are independent of this table: a parameter *borrows* its argument
and never copies it, whatever the category ([Borrowing](#borrowing)).

## Equality and identity

`==` compares by content for every value-carrying type — primitives, `string`,
`struct`, `enum`, `Range`, `List`, `Map`, `Set`, and tuples (element-wise,
equal arity) — as specified in
[`expressions/operators.md`](expressions/operators.md). Identity-carrying
values — function values, `Task<T>`, `View`, and `Error` — are compared by
identity: `==` is `true` exactly when both operands denote the same value.

- Identity is stable under `copy`: copying an identity-carrying value yields
  the same value, so `copy t == t` is `true`. `copy` deepens content; it never
  re-identifies what has no content.
- Structural comparison of reference values follows the references and
  compares the contents. Comparison of cyclic structures terminates: a pair of
  values already being compared is treated as equal rather than recursed into
  again, so `==` always halts.
- A comparison whose operands have unrelated types is a compile-time error
  (`E0211`), not an identity question.

## Ownership with `move`

`move` transfers ownership: it produces the value of its operand and
invalidates the binding it came from. From that point on the binding has been
**moved out**, and any use of it is a compile-time error (`E0403`).

`move` and `copy` are prefix operators at the `unary` precedence level, exactly
as [`expressions/operators.md`](expressions/operators.md) states, so `move a +
b` groups as `(move a) + b` and the second restriction below then rejects it.

- The operand of `move` MUST be a path naming a binding. `move p.x`,
  `move xs[0]`, and `move compute()` are errors (`E0403`): a transfer moves a whole
  binding, never a part of one.
- The binding MUST be one the current frame owns: a `let`, `let mut`,
  `const`, loop, or pattern binding. A parameter or receiver borrows its
  argument and owns no storage to transfer, so `move p` where `p` is a
  parameter is an error (`E0403`).
- `move` and `copy` are accepted in exactly three positions: the initializer
  of a `let`, an argument of a call, and the operand of `return`. In any other
  position — a condition, a binary operand, a field initializer, a literal
  element — they are a compile-time error (`E0403`).
- `move` applies to every category. For a value-semantic value the transfer is
  not observable through other bindings (there are none — the value was not
  shared); for a reference value it is: after `move`, no binding of the moving
  scope denotes that value.

```xulo
fn consume(data: List<int>): int {
  data[0] * 2
}

fn main() {
  let items = [1, 2, 3]
  let n = consume(move items)   // ownership transfers to the call
  print(n)                      // 2
  // print(items)               // error: `items` has been moved (E0403)
}
```

```xulo
fn steal(p: List<int>): int {
  let owned = move p            // error[E0403]: `p` is a borrow, not an owned binding
  owned[0]
}
```

The one place where `move` is *required* is capture into a `spawn` block
([`concurrency.md`](concurrency.md)): an ordinary enclosing binding may be
mentioned there only as a `move` or `copy` capture.

## Ownership with `copy`

`copy` produces an independent deep copy of its operand's value: a fresh value
in which every reachable content-bearing part has been duplicated, so that
later writes through either copy are invisible to the other. `copy` accepts
any expression as its operand and, like `move`, appears only as a `let`
initializer, an argument, or a `return` operand.

- Deep copying follows references through `List`, `Map`, and `Set` values,
  through `struct` and `enum` fields, and through tuple elements, and duplicates
  their contents. Copying a cyclic
  structure terminates: a value already being copied is represented by the
  copy made for it, so the copy graph mirrors the original.
- Identity-carrying values — function values, `Task<T>`, `View`, `Error` — are
  not duplicated: the copy is the same value. There is no way to clone a task
  in flight or to split a closure's captured storage by copying it.
- `copy` of an already value-semantic value is redundant but legal; it yields
  an equal value.

```xulo
let alice = { name: "Alice", age: 30 }
let bob = copy alice      // an independent deep copy
```

## Borrowing

A parameter never receives its argument's storage; it *borrows* it for the
duration of the call — extended past the call's return when the call returns
a `Task`, as [Span end](#span-end) defines
([`types/function-types.md`](types/function-types.md)):

- `p: T` is an **immutable borrow** — the default. The callee may read the
  argument; it may not write through the parameter.
- `p: mut T` is a **mutable borrow** — exclusive write access for the
  duration of the call. A method receiver is written the same way: `self`
  borrows immutably, `mut self` mutably
  ([`functions.md`](functions.md)).

A parameter is an ordinary binding for every purpose but ownership: it MAY be
passed on, where it lends its borrow to the inner call for that call's
duration — an immutable parameter lends an immutable borrow, a `mut`
parameter a mutable one. Onward lending obeys the same rules as the original
call.

Borrowing obeys **aliasing XOR mutability**: at any moment a value is either
shared by any number of immutable borrows or held by at most one mutable
borrow, never both. Three rules make that checkable; in them, a place is
**used** when it is read, written, moved, or passed as an argument.

1. **A mutable borrow needs a mutable place.** An argument passed to a `mut`
   parameter — or supplied as a `mut self` receiver — MUST be a place backed
   by a `mut` binding. Passing a literal, a computed value, or an immutable
   binding is a compile-time error (`E0404`).
2. **One call borrows without overlap.** Two arguments of one call MUST NOT
   denote overlapping places when either of the two parameters is `mut`;
   using a place this way is a compile-time error (`E0404`). When two places
   overlap is defined under [Overlap](#overlap).
3. **Borrows span suspension.** A borrow begins when the call starts and ends
   when the callee returns — or, when the call returns a `Task`, when that
   task settles ([Span end](#span-end)). While a **mutable** borrow of a
   place is outstanding, code outside the callee MUST NOT use the place at
   all (`E0405`). While an **immutable** borrow is outstanding, code outside
   the callee MUST NOT write the place (`E0411`) and MUST NOT start a
   mutable borrow of it (`E0404`); reads remain unrestricted. The same spans
   apply to a `let mut` binding written by an `async` closure that has
   suspended: the binding is borrowed by the running body until the
   closure's task settles.

```xulo
struct Counter { value: int }

fn bump(p: mut Counter) {
  p.value = p.value + 1
}

fn main() {
  let mut c = Counter(value: 0)
  bump(c)                    // mutable borrow of c for the call
  print(c.value)             // 1: the borrow ended with the call
}
```

```xulo
async fn fill(p: mut List<int>) {
  p[0] = 7
}

async fn main() {
  let mut xs = [0]
  let t = fill(xs)           // exclusive borrow of xs, until t settles
  // print(xs[0])            // error: `xs` is mutably borrowed (E0405)
  await t
  print(xs[0])               // 7: the borrow ended when t settled
}
```

```xulo
async fn peek(xs: List<int>) {
  print(xs[0])
}

async fn watch(mut xs: List<int>) {
  let t = peek(xs)           // immutable borrow of xs, until t settles
  // xs[0] = 9              // error[E0411]: written during an immutable borrow
  print(xs[0])               // ok: reads remain unrestricted
  await t                    // the borrow ends here
  xs[0] = 9                  // ok
}
```

Many immutable borrows may be outstanding at once, and ordinary reference
sharing between bindings is not a borrow and is always allowed.

### Overlap

Two places overlap unless, walking their paths from the base, the first
position where they differ proves them disjoint:

- different field names — `a.x` and `a.y` — or, in tuples, different
  positions — `p.0` and `p.1`;
- the same list subscripted by two distinct integer literals — `xs[0]` and
  `xs[1]`.

Any pair whose paths do not diverge this way overlaps: the same binding; one
path a prefix of the other (`a` and `a.x`); the same field (`a.x` and
`a.x.y`); subscripts that are not two distinct literals (`xs[i]` and `xs[j]`,
or `xs[i]` and `xs[0]`). Overlap is a static judgment: two non-literal
indices are never assumed distinct.

```xulo
fn pair(a: mut int, b: int): int { a + b }

fn overlap(mut xs: List<int>, i: int) {
  pair(xs[0], xs[1])         // ok: distinct literal subscripts are disjoint
  pair(xs[0], xs[0])         // error[E0404]: the same place, mutably borrowed twice
  pair(xs[0], xs[i])         // error[E0404]: a non-literal index is never assumed distinct
}
```

### Span end

A borrow from a call that returns a `Task<U>` — an `async` call, an `async`
closure call, or a synchronous call whose result type is `Task<U>` — lasts
until the checker can see that task settle. It can see it where control flow
reaches a point after an `await` whose operand contains the task's binding:
`await t`, `await Task::all([t, u])`, and `await Task::timeout(t, ms)` each
wait for `t`. When the scope contains no such `await`, the borrow lasts until
the scope ends.

- The synchronous case is conservative: the callee MAY have stored a borrowed
  place inside the task it returned, so the checker cannot end the borrow at
  the call even though the callee has returned; with no `await` of the task,
  the borrow lasts to the scope end, by which time the scope's exit has
  settled the task ([`concurrency.md`](concurrency.md)).
- Flow decides: on a path that has not reached such an `await` — the other
  branch of an `if`, an earlier loop iteration — the borrow is still
  outstanding.

```xulo
async fn fill(p: mut List<int>) {
  p[0] = 7
}

async fn main() {
  let mut xs = [0]
  let t = fill(xs)                    // mutable borrow of xs
  await Task::all([t])                 // ok: the await's operand contains `t`
  xs[0] = 1                           // ok: the borrow has ended
}
```

```xulo
async fn main() {
  let mut xs = [0]
  let t = fill(xs)                    // mutable borrow of xs
  // xs[0] = 1                       // error[E0405]: the borrow is outstanding
  await t                             //   until this await is reached
  xs[0] = 1                           // ok
}
```

## Mutability rules

`mut` belongs to a binding or a parameter, never to a type: `let mut x = 1`
binds an `int`, `p: mut Counter` declares a mutable borrow of a `Counter`
([`statements/let-and-assignment.md`](statements/let-and-assignment.md)).

- A place is **mutable** when the binding at the base of the path is a
  `let mut`, a `mut` parameter, or the `mut self` receiver of the running
  call; a field or subscript inherits the mutability of its base. Everything
  else is immutable, and assigning to it is a compile-time error (`E0401`).
- Value-semantic storage belongs to one binding, so mutation through a `mut`
  place changes only that binding's copy. Reference-semantic storage is
  shared, so a mutation through any mutable place naming it is visible
  through every binding naming that value.
- A `const` is immutable and has no mutable form; a shadowing `let mut` of
  the same name is an ordinary new binding.
- Across tasks, mutation of one value is permitted only through the shared
  state discipline — `shared` bindings and `lock` — specified in
  [`concurrency.md`](concurrency.md). The borrow rules above govern a single
  running task.

## Lifetimes (informal)

Xulo has no lifetime annotation; storage outlives every use that can be
written. The word *lifetime* is used here only to state when something is
usable:

- A binding's usable life runs from its declaration to the end of its block
  ([`names.md`](names.md)); using it earlier or after a `move` is a
  compile-time error (`E0403`).
- A borrow lives exactly as long as its call — or, when the call returns a
  `Task`, until that task settles, as [Span end](#span-end) defines — and no
  use outside the callee can name it, because borrows are parameters, not
  values that can be stored.
- A closure extends the life of what it captures: captured bindings become
  part of the closure's value and their storage remains valid for as long as
  the closure is reachable, even beyond the frame that created them
  ([`expressions/closures.md`](expressions/closures.md)).
- Because storage is reclaimed only when it is unreachable, no program can
  hold a reference to freed memory: dangling references do not exist, and
  neither use-after-free nor double-free is a possible failure.

## Garbage collection

Storage is managed automatically. Memory becomes eligible for reclamation
when it is unreachable from any root — a live binding, a running or suspended
call frame, a closure, a task, or a shared region — and is reclaimed at an
unspecified time thereafter.

- A value reachable from anywhere is kept: a returned closure keeps its
  captures alive, a suspended task keeps its frame alive, and a shared region
  lives while any binding or task names it.
- There is no explicit deallocation, no destructor, and no operation that
  frees memory early; a program cannot observe reclamation, and no program
  behavior depends on when it happens.
- The reclamation mechanism is not fixed: an implementation MAY reclaim
  storage by tracing, by reference counting together with a means of
  reclaiming cyclically reachable storage, by region-based bulk reclamation,
  or by any combination. A mechanism that never reclaims a value that has
  become unreachable from every root does not conform.
- Allocation strategy is likewise free, and the language already bounds the
  useful cases: the tree of a run that a later pass has replaced, and the
  frame of a settled task, cannot be reached by any program afterwards. An
  implementation MAY allocate such values in a region reclaimed in bulk when
  that boundary is crossed, and SHOULD keep any single reclamation pause
  within limits it documents. Neither the strategy nor the pause is
  observable.
- Deterministic *release of a resource* — a file, a connection — is a library
  concern expressed with ordinary control flow: cleanup is written on each
  path that needs it or registered once with `defer`
  ([`statements/return-and-block.md`](statements/return-and-block.md)),
  because the language provides no destructor. The language itself manages
  only storage.

## Runtime failures

A **runtime failure** stops a computation after the program has started. The
terms *runtime error* and *runtime trap* used elsewhere in this specification
mean the same thing. It is a panic
([`error-handling.md`](error-handling.md)): the program stops where the
condition occurs, and there is no handler for it.

| Condition | When it occurs | Specified in |
|-----------|----------------|--------------|
| List index out of bounds | reading or writing `xs[i]` where `i` is negative or at or past the list's length | [`types/composite-types.md`](types/composite-types.md) |
| Absent map key | reading `m[k]` for a key with no binding | [`expressions/path-and-access.md`](expressions/path-and-access.md) |
| String index out of range | `stringCodeAt` with an index outside `0 … stringSize(s) − 1`, or `stringSlice` with bounds outside `0 … stringSize(s)` or with `start > end` | [`builtins/prelude.md`](builtins/prelude.md) |
| Division or remainder by zero | `/` or `%` on non-constant operands whose value is zero | [`expressions/operators.md`](expressions/operators.md) |
| Shift amount out of range | `<<` or `>>` whose amount is negative or at or past the operands' bit width, on non-constant operands | [`expressions/operators.md`](expressions/operators.md) |
| Arithmetic overflow | an `int` or fixed-bit operation whose result exceeds its type, on non-constant operands | [`types/primitive-types.md`](types/primitive-types.md) |
| `Math::toFixed` digits out of range | `Math::toFixed` is called with `digits` outside `0...=15` | [`builtins/intrinsic-functions.md`](builtins/intrinsic-functions.md) |
| Unprovided environment key | reading an `@Environment` key no provision supplies | [`components/environment.md`](components/environment.md) |
| Stack or task exhaustion | recursion depth or the runtime's task limit is exceeded | [`functions.md`](functions.md) |

[`concurrency.md`](concurrency.md) defines two further runtime failures of its
own — misuse of `lock` and exhaustion of the task limit while spawning — and
[`error-handling.md`](error-handling.md) repeats this table alongside `panic`.

## Exit status

The language has no I/O and no `exit` function; how a program ends is a
property of the program and the host environment
([`README.md`](README.md), § 1.1).

- A program that terminates normally — `main` returns `unit`, the task
  returned by an `async` `main` settles, or the presentation target of a
  `View` program closes — exits with status **0**.
- A runtime failure that ends the program — a panic, and with it any
  condition of the closed tables above — exits with a **nonzero** status.
  The exact nonzero values, and how the failure is presented to the
  environment, are implementation-defined
  ([Implementation-defined](#implementation-defined)).

## Determinism

What this specification fixes, it fixes for every conforming program:

- Evaluation order: left-to-right for operands, arguments, and literal
  elements; source order for statements; declaration order for module-level
  initializers.
- Propagation: a `?` evaluates its operand exactly once and either yields its
  success value or performs the early return, abandoning the rest of the body
  at that point ([`error-handling.md`](error-handling.md)).
- Collection order: `Map` entries iterate in insertion order, and
  `Task::all` yields results in argument order
  ([`expressions/async-expressions.md`](expressions/async-expressions.md)).
- Termination of the structural operations: `==` and `copy` halt on cyclic
  values.
- Scope completion: a block finishes only after the tasks started inside it
  have settled ([`concurrency.md`](concurrency.md)).

What is deliberately not fixed: which ready task runs next, which of several
racing tasks settles first, how long a suspended task waits, and when
storage is reclaimed. Programs are deterministic when they do not depend on
these — that is, when they do not race.

## Undefined and implementation-defined behavior

**This specification defines no undefined behavior.** For every program that
passes the static checks, the meaning of each construct is fixed: evaluation
order and results are the ones given under [Determinism](#determinism) above,
and every condition the specification defines that can stop a computation at
run time is one of the runtime failures in the tables of this chapter and of
[`concurrency.md`](concurrency.md) — a specified failure with a specified
cause, after which the computation has ended. At no point is an
implementation permitted to continue in a way of its own choosing, and no
operation has a result, order, or effect that the specification leaves open
to it. One boundary is outside this promise, and is the only one: during an
extern call declared in [`ffi.md`](ffi.md), what the foreign code itself
does — memory it touches, values it produces, effects it has on the host —
is neither specified nor checked here. The Xulo side of that call remains
inside the promise: every argument expression, every conversion to an
ABI-safe type, and the result conversion are fully specified, and an
implementation MUST document which ABI document its registered tags denote
(below). Nothing else in the language crosses this boundary. Programs
rejected at compile time are rejected with one of the
diagnostics in [`type-system/errors.md`](type-system/errors.md).

Everything not fixed falls into exactly two categories, and the lists below
are closed: a construct not named in them is fully specified.

### Implementation-defined

Choices an implementation MAY make but MUST document — four, all at or
beyond the boundary between the program and its environment:

- the recursion and live-task limits behind *Stack or task exhaustion* and
  *Task exhaustion*, and whether anything outside the language can raise
  them;
- how exhaustion of resources the language does not model — available memory
  above all — is detected and reported;
- how a runtime failure is presented to that environment — message text,
  stream, and the exact nonzero exit status — since the language itself has
  no I/O ([`README.md`](README.md), § 1.1);
- which ABI document each registered tag (`"C"`, `"system"`) denotes on the
  target platform — and therefore the pointer width `W` that `usize` and
  `isize` have, and the layout every ABI-safe signature is laid out in
  ([`ffi.md`](ffi.md), [`types/primitive-types.md`](types/primitive-types.md)).

### Unspecified

Behavior fixed neither by this specification nor by any requirement to
document it. A program that depends on it is still well-defined — it is
nondeterministic, not ill-formed — but its outcomes are not portable:

- which ready task runs next, which of several racing tasks settles first,
  how many run concurrently, and how long a suspension lasts
  ([`concurrency.md`](concurrency.md));
- the iteration order of `Set<T>`
  ([`types/composite-types.md`](types/composite-types.md));
- when storage is reclaimed and where values are laid out — unobservable in
  any case, since no operation observes an address or a reclamation.
