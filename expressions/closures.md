# Closures

Functions are first-class values in Xulo: they may be passed, stored, returned,
and called through any expression. A **closure** is a function value written in
expression position, and it may use the bindings of the scope in which it was
written. This chapter defines closure syntax, closure types, capture, and the
higher-order functions built on them; `async` closures are introduced at the
end and specified in [`async-expressions.md`](async-expressions.md). The
expression layer as a whole is mapped in [`README.md`](README.md).

## Syntax

There are two closure forms. Both are expressions and both produce a value of
function type.

**Function expressions** reuse the shape of a function declaration, without a
name:

```xulo
let double = fn(x: Int): Int { x * 2 }
let one = fn(): Int { 1 }

print(double(21))          // 42
print(one())               // 1
```

**Arrow closures** consist of parameters, an optional result annotation, `=>`,
and a body that is either an expression or a block:

```xulo
let double = (x: Int) => x * 2                // result type inferred: Int
let add: fn(Int, Int): Int = (x, y) => x + y  // types from the expected type
let nothing = () => print("hi")               // no parameters: `()` is required
let classify = (n: Int): String => {          // block body
  if n > 0 { "positive" } else { "not positive" }
}
```

- A parameter list of one parameter MAY be written without parentheses:
  `x => x * 2`. Any other parameter list — including the empty list — MUST be
  parenthesized, so a closure without parameters is written `() => ...`.
- Parameter type annotations MAY be omitted when an expected type determines
  them: the closure then appears where a function type `fn(A, B): R` is
  expected — as a call argument, as the initializer of a typed binding, or as
  the operand of a typed `return`. The expected parameter types are used for
  the parameters.
- Parameter type annotations are REQUIRED when no expected type is available,
  as in `let double = (x: Int) => x * 2`.
- The result annotation `: R` is optional. When it is present it MUST agree
  with the type of the body; when it is omitted, the result type is the type of
  the body's trailing expression, or `Unit` if the block has none.
- A `return` operand inside a closure body returns from the closure and MUST
  have the closure's result type.
- The `async` variants are `async fn(x: T): U { ... }` and
  `async (x: T): U => ...` (see [`async-expressions.md`](async-expressions.md)).

## Type

The type of a closure is an ordinary function type `fn(A, B): R` (see
[`function-types.md`](../types/function-types.md)); a closure and a named
function with the same parameters and result are interchangeable:

```xulo
fn apply(f: fn(Int): Int, x: Int): Int {
  f(x)
}

print(apply(fn(x: Int): Int { x * 3 }, 7))   // 21
```

- Parameter and result types are taken from the annotations and from the
  expected type as described above; nothing else about a closure's type is
  inferred.
- Closures are values: they may be assigned to bindings, stored in `List` and
  `Map` values, passed as arguments, and returned from functions.
- A closure expression introduces no name for itself. Its body MUST NOT refer
  to the closure being defined; recursion and mutual recursion are expressed
  with named `fn` declarations (see [`functions.md`](../functions.md)).

## Capture rules

A closure captures every binding from an enclosing scope that its body
references. Capture is **by reference**: the closure shares the captured
binding's storage instead of copying the value.

- Reading a captured immutable binding (`let`, `const`, or a parameter) is
  unrestricted for as long as the closure value exists.
- A closure MAY assign to a captured binding only when that binding was
  declared `let mut`. Assigning to a captured non-`mut` binding is a
  compile-time error, exactly as it is anywhere else (see
  [`let-and-assignment.md`](../statements/let-and-assignment.md)).
- While a binding and a closure that captures it are both alive, a mutation
  made through the closure is visible through the binding, and a mutation made
  through the binding is visible through the closure.
- A closure MUST NOT capture a binding whose value has been transferred with
  `move`; referring to such a binding in a closure body is a compile-time error
  (see [`memory-and-runtime.md`](../memory-and-runtime.md)).
- The loop variable of a `for` loop is a fresh binding for each iteration, so a
  closure created during an iteration captures that iteration's binding and no
  other (see [`control-flow.md`](control-flow.md)).

```xulo
fn makeAdder(n: Int): fn(Int): Int {
  (v: Int) => v + n            // read capture of `n`
}

fn makeCounter(): fn(): Int {
  let mut count = 0
  fn(): Int {
    count = count + 1          // mutable capture of `count`
    count
  }
}

let add5 = makeAdder(5)
print(add5(10))                // 15

let counter = makeCounter()
print(counter())               // 1
print(counter())               // 2
```

## Capture modes

Xulo has exactly two capture modes, chosen by the captured binding itself:

| Mode | When | Effect |
|------|------|--------|
| read capture | the body only reads the binding | the closure shares the binding's value |
| mutable capture | the body assigns to a binding declared `let mut` | the closure shares the binding's storage; writes are visible to both sides |

- There is no move-capture form: a closure parameter list contains only
  parameter names and optional types, never a `move` modifier. Ownership
  transfer is expressed by the `move expr` operator
  ([`memory-and-runtime.md`](../memory-and-runtime.md)).
- The set of captured bindings is implicit: there is no capture-list syntax,
  and a binding that the body never mentions is not captured.

## Lifetime of captured state

A closure value may outlive the call frame in which it was created. Captured
bindings become part of the closure value and remain valid for as long as the
closure exists:

- storage for a captured binding is retained by any closure that captures it,
  so a function may return a closure that uses its own locals — `makeCounter`
  above returns a closure over `count` after `makeCounter` has returned;
- the enclosing scope and the closure share that storage while both are alive,
  which is what makes mutations through the closure visible to the binding;
- once neither the binding nor any closure capturing it is reachable, the
  storage is released.

The storage model behind this sharing is specified in
[`memory-and-runtime.md`](../memory-and-runtime.md).

## Immediately-invoked closures

A closure may be invoked at the point where it is written. The closure is
parenthesized before the call:

```xulo
let n = ((x: Int): Int => x + 1)(41)    // 42
```

Parenthesization makes the extent of the closure explicit; without it the call
would be parsed as part of the closure's body. An immediately-invoked closure
is useful for scoping a computation that needs captures but not a name.

## Higher-order functions

A function that takes or returns function values is a **higher-order
function**. Generic list operations are written with `List<T>` and function
parameters (see [`generics.md`](../types/generics.md)):

```xulo
fn map<T, U>(xs: List<T>, f: fn(T): U): List<U> {
  let mut out: List<U> = []
  for x in xs {
    out = out + [f(x)]
  }
  out
}

fn filter<T>(xs: List<T>, keep: fn(T): Boolean): List<T> {
  let mut out: List<T> = []
  for x in xs {
    if keep(x) {
      out = out + [x]
    }
  }
  out
}

fn fold<T, A>(xs: List<T>, init: A, step: fn(A, T): A): A {
  let mut acc = init
  for x in xs {
    acc = step(acc, x)
  }
  acc
}

let xs = [1, 2, 3, 4]
let doubled = map(xs, x => x * 2)          // expected type supplies `x`
let evens = filter(xs, x => x % 2 == 0)
let total = fold(xs, 0, (acc, x) => acc + x)
```

Function composition is itself a higher-order function:

```xulo
fn compose<A, B, C>(f: fn(A): B, g: fn(B): C): fn(A): C {
  (a: A) => g(f(a))
}

let transform = compose((x: Int) => x + 1, (x: Int) => x * 2)
print(transform(20))    // 42
```

## Closures vs named functions

- Use a closure for short, local behavior — an argument passed inline, a
  callback, a computation that needs the surrounding captures.
- Use a named `fn` declaration for anything that needs a name of its own, is
  called from several places, recurses, or is exported with `pub`
  (see [`functions.md`](../functions.md)).
- The two are otherwise identical: both have a `fn(...)` type, and mentioning a
  named function by name yields a function value of that type.

## Async closures

An `async` closure is written by prefixing either form with `async`:
`async (x: T): U => ...` or `async fn(x: T): U { ... }`. Calling it returns a
`Task<U>` instead of `U`, and the result is obtained with `await`:

```xulo
async fn main() {
  let work = async (x: Int): Int => x * 2
  print(await work(21))    // 42
}
```

`async` functions, `await`, tasks, and their evaluation semantics are specified
in [`async-expressions.md`](async-expressions.md).
