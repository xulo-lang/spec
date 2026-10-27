# Path and Access — Delta: Array Subscripts

> **Delta** — assumes `change-collection-members` (the List-only subscript)
> and replaces the “Subscript `[]`” section of
> `expressions/path-and-access.md` with the List-and-Array version; the rest is
> unchanged.

## Subscript `[]`

`expr[index]` reads an element of a `List<T>` or an `Array<T, N>`.

```xulo
xs[0]          // list element
a[0]           // array element
```

The index MUST have type `int` and the result is `T`. A subscript on a `Map`,
on a `string`, or on any other type is a compile-time error (`E0215`); a map
key is read with `m.get(k)`, and a string is inspected through its members —
`.len`, `.codeAt(i)`, `.slice(start, end)`, `.byteLen`, `.bytes()`.

Element access yields the element type directly — there is no automatic
optional. Reading past the end of a `List` or `Array` (a negative index
included) is a runtime error as defined in
[`../memory-and-runtime.md`](../memory-and-runtime.md); a lookup that may be
absent is not a subscript.

Assigning to a subscript writes an element: the base MUST be a place
expression backed by a mutable binding, the index MUST be an `int`, and the
value MUST have the element type. Writing outside the bounds of the target is
a runtime error.

```xulo
let mut xs = [1, 2, 3]
xs[0] = 9
let mut a: Array<int, 3> = [1, 2, 3]
a[0] = 9
```
