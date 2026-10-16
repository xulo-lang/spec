# Paths and Access — Delta: The Two Lanes

> **Delta** — replaces the introduction and the **Member access**, **Enum
> variant paths**, and **Qualified calls** sections of
> `expressions/path-and-access.md`. **Identifiers as paths**, **Positional
> access**, **Subscript**, **Optional chaining**, and **Static and instance
> separation** are unchanged.

Access expressions name things and reach into them: a bare identifier names a
binding, `::` walks the declaration lane — paths, packages, modules, variants,
associated names — `.` reaches into a value, `[]` indexes a collection, and
`?.` performs an access that tolerates `null`. This chapter defines each form,
its type, and its errors.

## Member access `.`

`expr.name` reads a field or invokes a method — `.` reaches into a value,
never into a namespace:

```xulo
user.name            // field of a struct, or an entry of a map
r.area()             // method declared in an `impl` block
getBox().width       // member access through a call result
```

Member access is a postfix operator of the highest precedence, so it groups
from the left: `a.b.c()` selects `b` from `a`, then selects `c` from that
result, and then calls it.

The type of a field access is the declared type of that field. The type of a
method access is the method's function type with `self` bound to the
receiver (see [`../functions.md`](../functions.md)); the call itself is
specified in [`calls.md`](calls.md).

When the receiver has type `Map<String, V>`, `expr.name` is the **member form
of a map entry**: it is exactly `expr["name"]` — type `V`, and a missing key
is the same runtime error as the subscript read of that key. On a map whose
key type is not `String` no member form exists
([`../types/composite-types.md`](../types/composite-types.md)).

A member that does not exist on the receiver's type is a compile-time error (`E0102`)
(see [`../type-system/errors.md`](../type-system/errors.md)). Members are
private by default, so a member that is not `pub` is accessible only inside
the module that declares it (see [`../modules/README.md`](../modules/README.md)).

## Paths `::`

`::` walks names: a path from a head through segments, one syntax for every
qualified name.

```xulo
Theme::Dark              // enum variant
form::validate           // module member
lsp-server::run          // package member
self::cache              // current module member
super::fields            // parent module member
Math::PI                 // built-in namespace member
Area::area               // explicit trait dispatch
Shape::Circle(2)         // variant construction: path plus arguments
```

A path head is a package name (through the manifest mapping), a root module
of this package, `self`, or `super`; the head and its resolution — including
the hyphen rule that keeps `lsp-server::run()` a path and
`total - Math::PI` subtraction — are specified in
[`../modules/import-export.md`](../modules/import-export.md).

- A variant without a payload denotes a value of the enum type directly; a
  variant with a payload is applied to exactly the arguments its declaration
  requires. The same paths appear in `match` patterns (see
  [`control-flow.md`](control-flow.md))
  ([`../types/enums.md`](../types/enums.md)).
- `.` is never used to name a variant, a namespace member, or a module
  member — `Theme.Dark` and `Math.PI` are not well-formed.
- **Paths need no prior `use`.** `form::validate(1, 2)` is callable the moment
  `form` resolves; `use form::validate` exists to bind the short name
  `validate`, not to grant access. Every segment a path walks must be visible:
  a private name anywhere in the chain is `E0602`, a missing one `E0601`.

The type of a path expression is the type of what its final segment resolves
to — a function's type, a constant's type, a variant's enum type, a built-in
namespace member's declared type. An unresolvable head is `E0607`, an
unresolvable later segment `E0101` or `E0601` according to whether the
previous segment was a local name or a module
([`../modules/import-export.md`](../modules/import-export.md)).

## Qualified calls

A call may be written through a path rather than through a bare name:

```xulo
Math::max(a, b)            // built-in namespace
Task::all(tasks)           // built-in namespace
Area::area(r)              // explicit trait dispatch
form::validate(1, 2)       // module member, no `use` required
lsp-server::router::run()  // dependency package head
```

`Trait::method(receiver)` is the explicit trait dispatch form and is required
where the receiver's type is a generic parameter bounded by that trait; it is
specified in [`../types/traits.md`](../types/traits.md). The call rules are
those of [`calls.md`](calls.md).
