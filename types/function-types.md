# Function Types

A function type describes a callable value: the types of its parameters and the type of its result. Functions are first-class in Xulo — named functions, closures, and methods are all ordinary values of a function type — and function types appear in parameter annotations, field types, and type aliases. This chapter specifies the syntax of function types, which values have them, how declarations affect them, and how they relate to each other.

## Syntax

A function type is written `fn(P1, P2, …): R`, listing the parameter types and the result type:

```text
FnType   = [ 'async' ] 'fn' '(' [ FnParams ] ')' [ ':' Type ] ;
FnParams = FnParam { ',' FnParam } ;
FnParam  = [ Identifier ':' ] Type ;
```

- Parameter names are optional: `fn(string): int` and `fn(name: string): int` are both well-formed. Parameter names in a function type are documentation — two function types that differ only in parameter names are the same type.
- A zero-parameter function type is `fn(): R`. The spelling `fn(unit): R` is *not* an alternative for it: `unit` is an ordinary parameter type, and a function that takes no arguments is written `fn(): R`.
- The result type may be omitted: `fn(string)` denotes the same type as `fn(string): unit`.
- Variadic parameters do not exist. Every function type has a fixed parameter list, and a call MUST supply exactly the parameters the type lists — except that a direct call to a *declaration* MAY omit trailing defaulted parameters (see [Parameters](#parameters)).
- `async fn(A): B` is notation for the type `fn(A): Task<B>` (see [async function types](#async-function-types)); either spelling MAY be used wherever that type is required.

## Values of function type

```xulo
fn add(a: int, b: int): int { a + b }

let f: fn(int, int): int = add      // a named function is a value
let g = fn(x: int): int { x * 2 }   // function literal
let h = (x: int): int => x * 3      // arrow form, same type

fn apply(f: fn(int): int, x: int): int { f(x) }

let xs = [fn(): int { 1 }, fn(): int { 2 }]   // function values in a list
```

- Every function declaration introduces a binding of its own signature type: `add` above has type `fn(int, int): int`. Its declared parameter types and result type are exactly its type as a value.
- Function values are assigned, passed as arguments, returned, stored in fields and collections, and called like any other value. A call of a function-typed expression is written `f(x, y)` and has the result type of the function type.
- **`null` is not a member of a function type.** Assigning `null` to a value of type `fn(A): B` is a compile-time error. A nullable function value must say so: `(fn(A): B)?` is the optional type over a function type, and reading it follows the optional rules in [`composite-types.md`](composite-types.md).

## Parameters

```xulo
fn greet(name: string = "stranger"): string {
  `Hello, ${name}`
}

fn area(rect: { w: int, h: int }): int { rect.w * rect.h }
fn grow(rect: mut { w: int, h: int }) { rect.w = rect.w + 1 }
```

- Parameter types are written in the declaration and are never inferred from the arguments: every argument expression MUST be assignable to the type of its parameter, and arguments are evaluated left to right. The one exception is a closure, which MAY omit an annotation when the expected type or the body determines it (see [`type-relations.md`](type-relations.md)).
- **Value and borrow modes.** A parameter written `p: T` receives an immutable borrow and `p: mut T` a mutable borrow; `move` and `copy` at the call site transfer ownership or deep-copy the argument. These rules are specified in [`../memory-and-runtime.md`](../memory-and-runtime.md) and [`../functions.md`](../functions.md).
- **Default parameters belong to the declaration, not to the type.** The function type of a function with default parameters lists *all* of its parameters: `fn greet(name: string = "stranger"): string` has type `fn(string): string`. A direct call MAY omit any trailing parameter that the declaration gives a default to; a call through a value of type `fn(string): string` MUST pass exactly one argument, because defaults are not recoverable from a function type.
- A trailing parameter of optional type may likewise be omitted at a direct call site, defaulting to `null`: a function declared `fn greet(name: string?): string` may be called as `greet()`.
- **Named arguments** belong to the declaration as well: a direct call MAY name its arguments (`Button(variant: "outline", label: "Submit")`); once one argument is named, every argument MUST be named, and order is free. A call through a value of function type uses positional arguments only. See [`../functions.md`](../functions.md).

```xulo
greet()                 // trailing default applied
greet("Ada")            // positional
greet(name: "Ada")      // named; all arguments named

let g: fn(string): string = greet
g("Ada")                // OK: the type lists exactly one parameter
g()                     // error: the type lists one parameter
```

## Closures

A closure expression is a value whose type is the function type written by its own signature:

```xulo
fn makeAdder(n: int): fn(int): int {
  fn(v: int): int { v + n }
}

let add5 = makeAdder(5)
print(add5(10))
```

- Both closure forms — the `fn(...)` function literal and the `(params): R => expr` arrow — have ordinary function types; wherever a parameter expects `fn(A): B`, either form MAY be passed.
- A closure captures the bindings of its enclosing scope. Its type depends only on its declared parameters and result type, never on what it captures. Capture and capture mutability are specified in [`../expressions/closures.md`](../expressions/closures.md).
- A closure is assignable to a function type exactly when a function of that signature is: same arity, assignable parameters, and an assignable result, with parameter names irrelevant. A closure whose parameters are unannotated takes its parameter types from the expected function type.
- A closure MAY be declared `async`, in which case its evaluated type is `fn(A): Task<B>` (see below).

## async function types

An `async` declaration writes the type that the *body* produces; the type of the value it produces is the corresponding `Task` type:

| Written form | Value's type | Body must produce |
|--------------|--------------|-------------------|
| `async fn foo(): T` | `fn(): Task<T>` | `T` |
| `async fn bar()` | `fn(): Task<unit>` | `unit` |
| `async (x: int): int => x + 1` | `fn(int): Task<int>` | `int` |

- The declared result type of an `async` function is the *evaluated* type: inside the body, `return` and the trailing expression MUST match `T`, and the value is wrapped as `Task<T>` before it leaves the function.
- `await expr` requires `expr` to have type `Task<T>` and yields `T`; it is legal only inside an `async` function or closure body. These rules are specified in [`../expressions/async-expressions.md`](../expressions/async-expressions.md).
- A non-`async` function MAY declare a `Task` result directly — `fn load(): Task<int>` — and is then an ordinary function that returns a task; it is not itself an async body and MUST NOT use `await`.

```xulo
async fn fetch(url: string): string { url }

fn load(): Task<int> { Task.resolve(1) }

let a: fn(string): Task<string> = fetch   // async function as a value
let b: fn(): Task<int> = load             // sync function returning a task
```

- `Task<T>` is a built-in generic type used as the result of an `async` function type; it requires no import. Its built-in utilities (`Task.all`, `Task.race`, `Task.resolve`, `Task.reject`) are specified in [`../expressions/async-expressions.md`](../expressions/async-expressions.md), and their concurrency model in [`../concurrency.md`](../concurrency.md).

## Function subtyping and higher-order types

Function types are compared structurally, with variance:

- **Parameters are contravariant**: `fn(A): R` MAY be used where `fn(A'): R'` is required whenever `A'` is assignable to `A` — a function that accepts *more* is usable where one that accepts *less* is expected.
- **Results are covariant**: `fn(A): R` MAY be used where `fn(A): R'` is required whenever `R` is assignable to `R'`.
- **Arity is exact**: types with different parameter counts are never compatible; there is no optional-argument widening at the type level.

```xulo
fn take_any(o: object): int { 0 }

fn run(g: fn({ name: string }): int): int {
  g({ name: "x" })
}

let n = run(take_any)   // OK: take_any accepts every object
```

Here `{ name: string }` is assignable to `object`, so `take_any` — which accepts more than `run` requires — satisfies the required type `fn({ name: string }): int`.

Function types may appear in any position, including as the parameter or result of another function type, so higher-order functions are written directly (`fn(fn(int): int): int`); the language has no separate notation for higher-rank polymorphism. A generic function used where a function type is expected MUST have its type parameters determined by the expected type or the arguments at that point; otherwise a compile-time error is reported. The complete compatibility and subtyping rules are in [`type-relations.md`](type-relations.md).

## Method types

Methods are functions declared in an `impl` block or a `trait`. The first parameter of a method is its receiver.

```xulo
struct Rectangle { w: int, h: int }

impl Rectangle {
  fn area(self): int { self.w * self.h }
  fn grow(mut self) { self.w = self.w + 1 }
}

trait Area {
  fn area(self): int
}

impl Area for Rectangle {
  fn area(self): int { self.w * self.h }
}
```

- The receiver is written `self` — an immutable borrow of the receiver — or `mut self`, a mutable borrow; a method that assigns to `self` or its fields MUST declare `mut self`, and the receiver at the call site must be a mutable place (see [`../memory-and-runtime.md`](../memory-and-runtime.md)). The receiver of a trait method and of its implementation MUST use the same form.
- The receiver counts as the first parameter of the method's function type: the trait method `Area.area` above corresponds to `fn(Rectangle): int`. A call `r.area()` evaluates the receiver once, passes it as the receiver argument, and has the method's declared result type.
- **Method references are explicit paths.** The only way to reference a method as a value is the trait-qualified path `Trait.method`. Its type is the trait's method signature with `self` replaced by the receiver type, and the receiver type MUST be determined by the surrounding context — including a visible `impl` for that receiver type:

```xulo
fn main() {
  let measure: fn(Rectangle): int = Area.area
}
```

  An `Area.area` expression whose receiver type cannot be determined from context is a compile-time error.
- Member access without a call — `r.area` — denotes a field access; referring to a method that way is a compile-time error. Inherent methods are reached through a call (`r.area()`), trait methods through `Trait.method`.
- Trait declarations, `impl` blocks, and dispatch are specified in [`traits.md`](traits.md); method declarations and parameter modes in [`../functions.md`](../functions.md).
