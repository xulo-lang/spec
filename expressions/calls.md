# Calls

A call applies a function, method, closure, or component to arguments. The
call syntax is uniform — `callee(arguments)` — and the rules below define how
arguments are matched to parameters, in what order they are evaluated, and
what is an error.

## Call syntax

A call is a postfix operator: a callee followed by a parenthesized,
comma-separated argument list. A trailing comma is allowed, and an empty
argument list — `f()` — passes no arguments.

```xulo
greet("world")
math.max(a, b,)
```

```text
Call         = Callee '(' [ ArgumentList ] ')' ;
ArgumentList = Argument ( ',' Argument )* ','? ;
Argument     = Identifier ':' ( "$" Identifier | Expression ) (* named *)
             | "$" Identifier                              (* bound state *)
             | Expression                                  (* positional *) ;
```

For a declared `fn`, the number of arguments supplied MUST match the declared
parameter list once default and optional parameters are accounted for: every
parameter that is neither defaulted nor optional MUST receive an argument, and
no parameter MAY receive more than one argument. See
[`../functions.md`](../functions.md) for declarations.

## Positional arguments

Positional arguments are matched to parameters in declaration order. They are
evaluated strictly left-to-right, as specified in
[`README.md`](README.md) (§ Evaluation Order), and every argument is
evaluated exactly once:

```xulo
log("start", timestamp())
```

## Named arguments

A named argument writes the parameter's name followed by `:` and an
expression.

```xulo
Button(variant: "outline", label: "Submit")
```

Named arguments MAY be reordered relative to the parameter list. Each named
argument MUST name a parameter declared by the callee, and naming the same
parameter twice is an error. Positional arguments come first: once a named
argument appears, every following argument MUST be named as well. Mixing is
therefore one-directional:

```xulo
fn render(text: String, size: Int, bold: Boolean): View {
  Text(text)
}

render("Hi", bold: true, size: 14)   // OK: positional, then named
render(text: "Hi", 14, bold: true)   // error: positional after named
```

## Binding arguments

In a component invocation an argument MAY be the binding form `$name`, labeled
(`value: $name`) or bare:

```xulo
Input(value: $name)
```

`$name` is not an expression: it passes a binding — the state cell of the
named variable — rather than a value, it is legal only in an argument position
of a component invocation, and the name MUST be declared by `@State` or
`@Store` in the enclosing component body. The mechanism and its errors are
specified in [`../components/binding.md`](../components/binding.md).

## Default parameters

A parameter MAY declare a default with `=`.

```xulo
fn greet(name: String = "stranger"): String {
  "Hello, " + name
}
```

A default value is evaluated at each call that omits the argument, not once at
declaration, so a default may depend on values computed at call time. The
parameter's type MUST admit the default value: `name: String = 42` is a
compile-time error.

Any parameter MAY have a default. An omitted argument MUST be supplied by name
or MUST be trailing in the positional list: a positional call MAY omit only a
suffix of the parameters, and every parameter in that suffix MUST have a
default (or be optional, below). A call that leaves a required parameter
unfilled, or that fills a later parameter while skipping an earlier one, is an
error.

## Optional parameters

A parameter whose type is optional (`T?`) MAY be omitted at the call site, in
which case it is bound to `null`.

```xulo
fn greet(name: String?): String {
  if name != null { "Hello, " + name } else { "Hello, stranger" }
}

greet()      // name is null
greet("Ada")
```

An optional parameter without a default accepts `null` explicitly as well, so
`greet(Null)` and `greet()` are equivalent. Optional and defaulted parameters
follow the same positional-trailing or named rule stated above.

## Method calls

A method call writes `receiver.method(arguments)`.

```xulo
r.area()
r.is_square()
```

Resolution proceeds in this order: first the methods declared in `impl` blocks
for the receiver's own type, then methods provided by trait implementations
that are in scope at the call site (see
[`../types/traits.md`](../types/traits.md)). The explicit form
`Trait.method(receiver)` is always available and is required when the
receiver's type is a generic parameter bounded by a trait. The receiver is
supplied to the declared `self` parameter, which is not written at the call
site; argument matching, evaluation order, and arity otherwise follow the
rules above. A method call whose receiver is `null` is written with optional
chaining, `receiver?.method()`, which evaluates to `null` instead of
evaluating the call (see [`path-and-access.md`](path-and-access.md)).

## Generic calls

Type arguments of a generic function are inferred at the call site from the
actual arguments and from the expected type.

```xulo
fn first<T>(items: List<T>): T {
  items[0]
}

let n: Int = first([1, 2, 3])    // T is Int
let s = first(["a", "b"])        // T is String
```

Explicit type arguments at a call site are not part of the language: writing
`first<Int>(...)` is not well-formed. Inference, bounds, and unification are
specified in [`../types/generics.md`](../types/generics.md).

## Closures as callees

Any expression of function type may be called, including a variable holding a
closure, a list element, a subscript, and the result of another call.

```xulo
double(21)
xs[0](10)
getFn()(x)
(f)(5)
(fn(x: Int): Int { x })(1)
```

The callee is evaluated first, then the arguments left-to-right. A call made
through a value of function type (`fn(...)`) accepts positional arguments
only, and the count MUST match exactly — named arguments are not available
through function values, only through declared functions and methods. See
[`closures.md`](closures.md).

## Component calls

A component call looks like a function call whose declared return type is
`View`, and may be followed by a trailing block of children.

```xulo
Text("Hi")
Button("Go", variant: "primary") {
  Text("Confirm")
}
```

The first positional argument is conventionally the component's content or
label, named arguments are attributes, and a trailing block becomes the
component's children rather than an argument. Component calls follow the same
argument rules as function calls; the component layer — attributes, blocks,
and `$` binding (above) — is specified in
[`../components/view-syntax.md`](../components/view-syntax.md) and
[`../components/binding.md`](../components/binding.md).

## Recursion and forward references

A call requires its callee to be declared. Recursion is unrestricted: a
function may call itself. Module-level functions may also forward-reference
one another, so mutual recursion between functions in the same module is
well-formed without any ordering constraint (see
[`../names.md`](../names.md)). A call to an undeclared or private callee
outside its module is a compile-time error.

## Evaluation errors

Calling a value that is not of function type is a compile-time error; there
is no dynamic dispatch to an arbitrary value and no truthiness that would make
a non-function callable. Arguments MUST match the declared parameter types,
and a mismatch — including a missing required argument, an unknown named
argument, a duplicate argument, or an argument of an unrelated type — is
reported as a compile-time error (see
[`../type-system/errors.md`](../type-system/errors.md)). Failures that cannot
be detected statically, such as indexing past the end of a list inside the
call, are runtime errors (see [`../memory-and-runtime.md`](../memory-and-runtime.md)).
