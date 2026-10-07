# Functions

A function is a named, typed unit of computation declared with `fn`. Functions
are first-class values: a named function may be passed, stored, and returned
like any other value. This chapter defines function declarations, parameters,
return values, methods, trait methods, generic and async functions, and the
inference boundary. Function *types* are specified in
[`types/function-types.md`](types/function-types.md); anonymous function
values in [`expressions/closures.md`](expressions/closures.md).

## Definition

```xulo
fn log(message: string) {
  print(message)
}

fn add(a: int, b: int): int { a + b }
```

- The form is `fn name(p: T, q: U): R { … }`. Parameters are `name: type`
  pairs; the return type follows `:` after the parameter list.
- **Every parameter MUST be annotated with its type.** Parameter types are
  never inferred from the arguments.
- The return type is written when the function produces a value. A missing
  return type means `unit` ([`types/primitive-types.md`](types/primitive-types.md));
  declaring `: unit` explicitly is equivalent to omitting it.
- A named `fn` declaration MAY appear at module level or inside a block body.
  A nested function is local to its block and is not exported. The anonymous
  forms — `fn(…)` and `(…) => …` — are closures
  ([`expressions/closures.md`](expressions/closures.md)).
- `pub` on a `fn` marks it as exported from its module; see [Visibility](#visibility).

## Parameters

```xulo
fn greet(name: string = "stranger"): string { "Hello, " + name }
fn render(text: string, size: int, bold: boolean): View { Text(text) }

render("Hi", bold: true, size: 14)   // OK: positional, then named
render(text: "Hi", 14, bold: true)   // error: positional after named
render("Hi")                          // error: required parameters missing
```

- Parameters are **required** unless they declare a default value or have an
  optional type. Names follow the conventions of [`names.md`](names.md).
- A parameter MAY declare a default with `=`. The default is evaluated at
  each call that omits the argument, and the parameter's type MUST admit the
  default value.
- A parameter whose type is optional (`T?`) MAY be omitted at the call site
  and is then bound to `null`.
- **Named arguments** write the parameter's name followed by `:`. Positional
  arguments come first: once a named argument appears, every following
  argument MUST be named. Named arguments MAY be reordered. Omitted
  positional arguments form a trailing suffix of parameters that have
  defaults or optional types.
- Calls through a value of function type (`fn(...)`) accept **positional
  arguments only**; named arguments exist only for declared functions and
  methods.
- Parameter scope is the function body: a parameter is in scope from the
  start of the body to its end.

## Return values

Return values are either **implicit** — the trailing expression of the body —
or **explicit** — a `return` statement. Both forms and the rules of `return`
are specified in [`statements/return-and-block.md`](statements/return-and-block.md).

```xulo
fn add(a: int, b: int): int { a + b }              // implicit return
fn subtract(a: int, b: int): int { return a - b }  // explicit return
```

- **Recursion** is unrestricted: a function MAY call itself, at any depth its
  runtime allows. A function's name is in scope within its own body.
- **Mutual recursion** among module-level functions is well-formed regardless
  of declaration order: module-level `fn` declarations are in scope
  throughout the whole file ([`names.md`](names.md)).

## Scope and forward references

- A function body MAY call functions declared **later** in the file. Module
  level `fn`, `struct`, `enum`, `trait`, `type`, and `impl` declarations MAY
  reference each other in any order.
- Variables and constants may not be forward-referenced. Module-level `let`,
  `let mut`, and `const` are in scope only from the end of their declaration
  to the end of the file, and their initializers run in declaration order.
- A nested `fn` follows ordinary block-scope order: its name is in scope
  within its own body (self-recursion is legal) and, from the end of its
  declaration, in the enclosing block. Two nested functions do not
  forward-reference each other; mutual recursion among nested functions is
  therefore unavailable — write it at module level, or restructure into a
  single recursive function.

```xulo
fn main() {
  let a = helper()     // legal: helper may be declared later at module level
  print(a)
}
fn helper(): int { 42 }
```

## Closures vs named functions

A named `fn` introduces a name; a closure is an anonymous function value in
expression position. The two share the `fn(...)` type and are interchangeable
wherever a function value is expected. Use a closure for short, local
behavior; use a named `fn` for anything that needs a name of its own,
recurses, or is exported with `pub`. Capture rules and closure syntax are
specified in [`expressions/closures.md`](expressions/closures.md).

## Methods and `self`

```xulo
struct Counter { value: int }

impl Counter {
  fn increment(mut self) { self.value = self.value + 1 }
  fn get(self): int { self.value }
}

fn main() {
  let mut c = Counter(value: 0)
  c.increment()
  print(c.get())          // 1
  let maybe: Counter? = null
  print(maybe?.get())     // null: the call is not evaluated
}
```

- Methods are functions declared inside an `impl` block. The first parameter
  is the **receiver**, written `self` (an immutable borrow) or `mut self`
  (a mutable borrow). `self` is an explicit parameter, never implicit. A
  method that assigns to `self` or a field of `self` MUST declare `mut self`.
- At the call site the receiver is not written as an argument:
  `c.increment()` supplies `c` to the declared `self` parameter.
- **Method resolution order:** `value.method(…)` resolves first to an
  inherent method declared in an `impl` block for the receiver's type, and
  otherwise to a trait method provided by an implementation in scope at the
  call site. Exactly one candidate MUST result; ambiguity is an error and
  MUST be disambiguated with explicit dispatch `Trait.method(receiver)`.
  Explicit dispatch is always available.
- Inside a body bounded by `T: Trait`, both `x.method(args)` and
  `Trait.method(x)` are permitted.
- A method call whose receiver may be `null` is written with optional
  chaining, `receiver?.method()`; the expression evaluates to `null` instead
  of evaluating the call
  ([`expressions/path-and-access.md`](expressions/path-and-access.md)).
- The receiver is the first parameter of the method's function type; using a
  method as a value with the receiver bound is specified in
  [`expressions/calls.md`](expressions/calls.md).

## Trait methods

```xulo
trait Greet {
  fn greeting(self): string;
  fn farewell(self): string
}

struct Person { name: string }

impl Greet for Person {
  fn greeting(self): string { "hello " + self.name }
  fn farewell(self): string { "bye " + self.name }
}
```

- A `trait` declaration hosts method **signatures only**: parameters and a
  return type, with no body. A trait MUST NOT provide a default implementation.
- **Every trait method MUST declare a return type.** Unlike an ordinary `fn`,
  whose omitted return type means `unit`, a trait method may not omit it; a
  trait method that produces no value writes `: unit`.
- Method declarations inside a `trait` are separated by `;`, and a `;` after
  the final method is optional.
- The receiver form is `self` or `mut self`. The receiver used by an
  implementation MUST match the one the trait declares. Remaining parameters
  MUST be annotated.
- An `impl Trait for Type` block MUST provide **every** method the trait
  declares. Partial implementations do not exist, and an implementation MUST
  NOT declare methods the trait does not declare — additional behavior
  belongs in an inherent `impl` block.
- Signatures MUST match: parameter types, the return type, and the receiver
  form of each implementation method MUST be mutually assignable with the
  declared signature. The trait layer is specified in
  [`types/traits.md`](types/traits.md).

## Generic functions

```xulo
trait Area { fn area(self): float; fn perimeter(self): float }

fn first<T>(xs: list<T>): T { xs[0] }
fn areaOf<T: Area>(shape: T): float { Area.area(shape) }           // explicit dispatch
fn perimeterOf<T>(shape: T): float where T: Area { shape.perimeter() }  // member form

let n = first([1, 2, 3])        // T = int
let s = first(["a", "b"])       // T = string
// let bad: string = first([1, 2])   // error: T inferred as int
```

- A generic function declares a type parameter list `<T, U, …>` after its
  name. Type parameters are in scope over the signature and the body.
- **Bounds** restrict the type arguments a parameter accepts. They are
  written inline — `<T: Area>` — or in a `where` clause that follows the
  signature and precedes the body — `where T: Area`. The forms are
  equivalent; a parameter MUST NOT be bounded in both places. Multiple
  bounds on one parameter are joined with `+`.
- **Call-site inference.** Type arguments are never written at a call site;
  explicit type arguments such as `first<int>(…)` are not part of the
  language. Type arguments are inferred from the arguments and the expected
  type, and every inferred argument MUST satisfy the parameter's bounds.
- Inside a body bounded by `T: Trait`, values of type `T` expose the members
  the bound declares; `Trait.method(x)` and `x.method()` are both legal.

Bounds, inference, and nested-angle-bracket spelling are specified in
[`types/generics.md`](types/generics.md).

## Async functions

```xulo
async fn doubleAsync(n: int): int {
  await Task.resolve(n * 2)
}
async fn main() { print(await doubleAsync(21)) }   // 42
```

- An `async` function is declared with `async` before `fn` (after `pub` when
  present). The declared return type denotes the *declared* result: the
  function name has the evaluated type `fn(...): Task<T>`. An `async`
  function with no declared return type has the evaluated type `fn(): Task<unit>`.
- Calling an `async` function starts its body immediately; the body runs
  until it suspends at an `await` or completes. `await` unwraps `Task<T>` to
  `T` and is legal only inside `async` bodies.
- Inside an `async` body the trailing expression and every `return` operand
  MUST have the declared type `T`, not `Task<T>`; wrapping is automatic.
- The `Task` type, its utilities, and the full async model are specified in
  [`expressions/async-expressions.md`](expressions/async-expressions.md);
  scheduling in [`concurrency.md`](concurrency.md).

## Recursion and termination

- Recursion is a first-class feature: a function MAY call itself with no depth
  limit in the language. The language imposes **no static termination
  guarantee** and no static bound on recursion depth. Unbounded recursion MAY
  fail at runtime with an error when the execution stack or the runtime's task
  limits are exhausted; that failure is a runtime condition, not a
  compile-time diagnostic.

## Overloading

Xulo functions are **not overloaded**. Within one scope there is at most one
function declaration per name; a second declaration of the same name in the
same scope is a duplicate-declaration error ([`names.md`](names.md)).

```xulo
fn show(x: int) { print(x) }
fn show(x: string) { print(x) }   // error: duplicate name
```

Type-directed dispatch exists only through **trait bounds**: a generic
function bounded by `T: Trait` resolves a member call or a `Trait.method(x)`
call statically to one implementation. There is no overloading by parameter
type, no default methods, and no dynamic dispatch on the runtime type of a
value.

## Visibility

```xulo
pub fn add(a: int, b: int): int { a + b }
fn helper(n: int): int { n * 2 }   // private to this module
```

- A module-level `fn` is **private by default**: it may be used only inside
  the file that declares it.
- `pub` on a module-level `fn` makes it visible to importers; `pub` is the
  only mechanism that grants that visibility.
- A nested `fn` is local to its block; `pub` does not apply to it. Member
  visibility of methods is specified in [`modules/visibility.md`](modules/visibility.md).

## Inference boundary

- **Named function declarations require full annotation.** Every parameter
  type MUST be written, and the return type MUST be written whenever the
  function produces a value; only an omitted return type — meaning `unit` —
  is allowed to be missing. Named functions never take parameter types from
  their arguments, at module level or nested in a block.
- **Local `let` bindings infer** from their initializer; the annotation is
  optional whenever the initializer determines the type
  ([`statements/let-and-assignment.md`](statements/let-and-assignment.md)).
- **Closure parameters may omit annotations only when an expected function
  type supplies them.** A closure appears where a type `fn(A, B): R` is
  expected — as a call argument, as the initializer of a typed binding, or
  as the operand of a typed `return` — and then the expected parameter types
  are used. When no expected type is available, closure parameter
  annotations are REQUIRED
  ([`expressions/closures.md`](expressions/closures.md)).

```xulo
fn apply(f: fn(int): int, x: int): int { f(x) }
let double = (x: int): int => x * 2        // annotation required
print(apply(x => x * 3, 7))                // optional: expected type supplies it
```

## Example

A complete program combining a trait, methods, a generic bounded function,
a generic with a `where` clause, and a recursive function:

```xulo
trait Area {
  fn area(self): float;
  fn perimeter(self): float
}

struct Rect { w: float, h: float }

impl Area for Rect {
  fn area(self): float { self.w * self.h }
  fn perimeter(self): float { 2.0 * (self.w + self.h) }
}

fn fib(n: int): int {
  if n <= 1 { n } else { fib(n - 1) + fib(n - 2) }
}

fn areaOf<T: Area>(shape: T): float { Area.area(shape) }
fn perimeterOf<T>(shape: T): float where T: Area { shape.perimeter() }

fn main() {
  let r = Rect(w: 3.0, h: 4.0)
  print(areaOf(r))          // 12: generic bound, explicit dispatch
  print(perimeterOf(r))     // 14: where clause, member form through the bound
  print(fib(10))            // 55: recursion
}
```

`areaOf` and `perimeterOf` are checked once per instantiation against the
`Area` bound; `Rect` satisfies that bound through its `impl`. `fib` recurses
with no static bound. Both dispatch forms are permitted inside the bound.
