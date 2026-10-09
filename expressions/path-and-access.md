# Paths and Access

Access expressions name things and reach into them: a bare identifier names a
binding, `.` reaches a field, method, namespace member, or tuple position, `::`
names an enum variant, `[]` indexes a collection, and `?.` performs an access
that tolerates `null`. This chapter defines each form, its type, and its
errors.

## Identifiers as paths

A bare identifier is a path of length one. It resolves to a binding,
declaration, or type name according to the scoping and shadowing rules of
[`../names.md`](../names.md):

```xulo
let mut total = 0
total
```

The type of the expression is the type of the name it resolves to. An
identifier that resolves to nothing, or that names a type where a value is
required, is a compile-time error (see
[`../type-system/errors.md`](../type-system/errors.md)).

## Member access `.`

`expr.name` reads a field, invokes a method, or reaches into a namespace.

```xulo
user.name            // field of a struct, or an entry of a map
r.area()             // method declared in an `impl` block
Math.PI              // constant in a built-in namespace
Task.all(tasks)      // function in a built-in namespace
Time.now()           // function in a built-in namespace
getBox().width       // member access through a call result
```

Member access is a postfix operator of the highest precedence, so it groups
from the left: `a.b.c()` selects `b` from `a`, then selects `c` from that
result, and then calls it.

The type of a field access is the declared type of that field. The type of a
method access is the method's function type with `self` bound to the
receiver (see [`../functions.md`](../functions.md)); the call itself is
specified in [`calls.md`](calls.md). The type of a namespace member is its
declared type.

When the receiver has type `map<string, V>`, `expr.name` is the **member form
of a map entry**: it is exactly `expr["name"]` — type `V`, and a missing key
is the same runtime error as the subscript read of that key. On a map whose
key type is not `string` no member form exists
([`../types/composite-types.md`](../types/composite-types.md)).

A member that does not exist on the receiver's type is a compile-time error
(see [`../type-system/errors.md`](../type-system/errors.md)). Members are
private by default, so a member that is not `pub` is accessible only inside
the module that declares it (see [`../modules/README.md`](../modules/README.md)).

## Positional access `.0`

On a tuple, the "name" after `.` is the element's position, counted from
zero:

```xulo
let p = (10, "ten")
p.0             // 10, the first element
p.1             // "ten", the second element
pair().0        // positional access through a call result
```

- The receiver MUST have a tuple type, and the position MUST exist in it —
  `0 ≤ i < arity`. A receiver of any other type, or a position at or past
  the arity, is a compile-time error (`E0220`).
- The position is an integer literal without a sign: `p.0` is the whole
  operator, and ordinary field names never begin with a digit, so
  positional and member access never compete for the same spelling.
- The type of `p.i` is the `i`-th element type of the tuple. Reading is an
  ordinary place read; writing `p.0 = v` follows the same place rules as a
  field write and requires a mutable binding (see
  [`../memory-and-runtime.md`](../memory-and-runtime.md)).
- The `null`-tolerant form exists too: `p?.0` evaluates to `null` when `p`
  is `null`, and otherwise to the element — optional chaining below.

## Enum variant paths `::`

`Enum::Variant` names an enum variant, and `Enum::Variant(args)` constructs a
value of that enum with its payload:

```xulo
let theme = Theme::Dark
let c = Shape::Circle(2)
```

`::` separates an enum name from a variant, and is used for nothing else. `.` is
never used to name a variant — `Theme.Dark` is not well-formed — and `::` is
never used for fields, methods, or namespace members, which use `.`.
A variant without a payload denotes a value of the enum type directly; a
variant with a payload is applied to exactly the arguments its declaration
requires. The same paths appear in `match` patterns (see
[`control-flow.md`](control-flow.md)). See
[`../types/enums.md`](../types/enums.md).

## Subscript `[]`

`expr[index]` reads an element of a collection.

```xulo
xs[0]          // list element
counts[key]    // map value
```

The index type follows the collection: a `list<T>` is indexed by `int` and
yields `T`; a `map<K, V>` is indexed by `K` and yields `V`. Strings are not
indexable (see [`../types/primitive-types.md`](../types/primitive-types.md));
select a character by other means, such as the built-in string intrinsics.

Element access yields the element type directly — there is no automatic
optional. Reading past the end of a `list` (a negative index included), or
reading a `map` key that is absent, is a runtime error as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md); a lookup that may be
absent is therefore guarded by an explicit presence test before the subscript
is evaluated.

Assigning to a subscript writes an element: the base MUST be a place
expression backed by a mutable binding, the index MUST have the collection's
index type, and the value MUST have the element type. Writing outside the
bounds of the target is a runtime error.

```xulo
let mut xs = [1, 2, 3]
xs[0] = 9
```

## Optional chaining

`?.` performs a member access that tolerates `null`. When the receiver is
`null`, the whole chain — every following `?.` and every call in it —
evaluates to `null`, and no operand to the right of the first `?.` is
evaluated at all:

```xulo
let theme = session?.profile?.theme ?? Theme.Light
```

Here `session` is read first; if it is `null`, `profile` is never read, the
inner chain is never evaluated, the result is `null`, and `??` supplies
`Theme.Light`. If `session` is not `null`, `profile` is read, and the same
test applies to it. If `a` has type `T?` and `T` has a member of type `U`,
then `a?.b` has type `U?`, so an optional chain is always consumed with `??`,
a ternary, a `null` test, or a context that admits `null`. The same holds for
a position: if `q` has type `(int, string)?`, then `q?.0` has type `int?`.
Optional chaining combines with `??` exactly this way, and with method calls
as in `session?.refresh()`; the operator is specified in
[`operators.md`](operators.md).

## Qualified calls

A call may be written through a path rather than through a bare name. Built-in
namespaces, imported namespaces, and traits are all called this way:

```xulo
Math.max(a, b)            // built-in namespace
Task.all(tasks)           // built-in namespace
Area.area(r)              // explicit trait dispatch
math.add(1, 2)            // after `import * as math from "…"`
```

`Trait.method(receiver)` is the explicit trait dispatch form and is required
where the receiver's type is a generic parameter bounded by that trait; it is
specified in [`../types/traits.md`](../types/traits.md). Namespace members
reached through `import * as ns` behave exactly like members of a built-in
namespace, and the call rules are those of [`calls.md`](calls.md).

## Static and instance separation

Xulo has no implicit receiver: there is no `this`, and no expression silently
refers to a current instance. `self` occurs only as an explicitly written
first parameter of a method declaration, where it receives the value the
method was called on:

```xulo
impl Rectangle {
  fn area(self): float {
    self.w * self.h
  }
}

r.area()
```

In `r.area()` the receiver `r` is supplied to `self`; inside the body, `self`
is an ordinary name resolved by the rules above. A method body therefore never
sees a hidden binding, and static items — functions, constants, and namespace
members — are reached only through explicit paths. Method declarations,
`self` parameter modes, and dispatch are specified in
[`../functions.md`](../functions.md) and
[`../types/traits.md`](../types/traits.md).
