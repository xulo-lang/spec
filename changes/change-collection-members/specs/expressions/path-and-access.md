# Path and Access — Delta: Built-in Members and List-Only Subscripts

> **Delta** — replaces the “Member access” and “Subscript” sections of
> `expressions/path-and-access.md`, and the path-head sentence in
> “Paths `::`”; the other sections are unchanged.

## Member access `.`

`expr.name` reads a field or invokes a method — `.` reaches into a value,
never into a namespace:

```xulo
user.name            // field of a struct
counts.get("a")      // built-in Map member
r.area()             // method declared in an `impl` block
getBox().width       // member access through a call result
```

Member access is a postfix operator of the highest precedence, so it groups
from the left: `a.b.c()` selects `b` from `a`, then selects `c` from that
result, and then calls it.

The type of a field access is the declared type of that field. On a built-in
type, the member's declared type is the type of the access: `.len`, `.size`,
and `.byteLen` are read-only properties of type `int`, and a property is not a
place — assigning to one is `E0402`. A method access has the method's function
type with the receiver bound to the receiver (see
[`../functions.md`](../functions.md)); the call itself is specified in
[`calls.md`](calls.md). A name that is not in the receiver's member set is a
compile-time error (`E0102`).

The built-in collections and `string` have closed inherent member sets,
specified with their types
([`../types/composite-types.md`](../types/composite-types.md),
[`../types/primitive-types.md`](../types/primitive-types.md)). A program
cannot add an inherent member to a built-in type; a trait implementation for
one is an ordinary trait method. These members resolve before inherent
`impl` methods and trait methods. There is no map entry member form: on
`Map<K, V>`, `m.name` is a member lookup, never the entry `"name"`; a key is
read with `m.get("name")`.

## Subscript `[]`

`expr[index]` reads an element of a `List<T>`.

```xulo
xs[0]          // list element
```

The index MUST have type `int` and the result is `T`. A subscript on a `Map`,
on a `string`, or on any other type is a compile-time error (`E0215`); a map
key is read with `m.get(k)`, and a string is inspected through its members —
`.len`, `.codeAt(i)`, `.slice(start, end)`, `.byteLen`, `.bytes()`.

Element access yields the element type directly — there is no automatic
optional. Reading past the end of a `List` (a negative index included) is a
runtime error as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md); a lookup that may be
absent is not a subscript.

Assigning to a subscript writes an element: the base MUST be a place
expression backed by a mutable binding, the index MUST be an `int`, and the
value MUST have the element type. Writing outside the bounds of the target is
a runtime error.

```xulo
let mut xs = [1, 2, 3]
xs[0] = 9
```

### Paths `::` — path heads (excerpt)

A path head is a package name (through the manifest mapping), a root module
of this package, `self`, `super`, or a type name for a variant or an
associated member — `Theme::Dark`, `Map::fromEntries`; the head and its
resolution, including the hyphen rule that keeps `lsp-server::run()` a path
and `total - Math::PI` subtraction, are specified in
[`../modules/import-export.md`](../modules/import-export.md).
