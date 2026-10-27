# Composite Types — Delta: Fixed-Length Array

> **Delta** — adds `## Array<T, N>` after the List section of
> `types/composite-types.md`; the rest of the chapter is unchanged. This delta
> assumes the const-generic syntax from `add-const-generics` and the List
> member names from `change-collection-members`.

## Array<T, N>

`Array<T, N>` is a built-in **inline, fixed-length value type**: `T` is a type
parameter, `N` is a const parameter of type `int` (from `add-const-generics`),
and N MUST be non-negative; a concrete negative N is `E0814`. Every value
holds exactly N elements of `T`, laid out inline; the type is invariant in `T`
and exact in N.

- Binding, assignment, argument passing, and return copy the Array value;
  each of its N elements is copied by its own category (a `List` element is
  shared, a `struct` element is value-copied). An `Array` has no backing-store
  identity and no reference sharing.
- `a[i]` and `a[i] = v` require an `int` index in `0 … N − 1`; an index outside
  that range is a runtime failure. `for x in a` visits the elements in index
  order.
- `.len` is a read-only `int` property equal to N, O(1).
- The read, callback, and sort members are the List members supplied by
  `change-collection-members` — `.slice`, `.indexOf`, `.lastIndexOf`,
  `.includes`, `.join`, `.entries`, `.reverse`, `.sort`, `.forEach`, `.map`,
  `.filter`, `.find`, `.findIndex`, `.some`, `.every`, `.reduce` — with these
  differences:
  - `.slice` and `.filter` return `List<T>`; `.map` returns `Array<U, N>`
    because the length is preserved;
  - `.reverse` and `.sort` mutate the array; `sort` is stable and takes a
    comparator;
  - callbacks are synchronous, receive the element only, and cannot mutate
    the receiver (the ordinary borrow rules, `E0404`/`E0405`).
- There is **no** `.push`, `.pop`, `.shift`, `.unshift`, `.splice`, `.resize`,
  `.truncate`, or `.clear`: the length is N. A member outside this set is
  `E0102`.
- A list literal checked against an expected `Array<T, N>` is an array
  literal: its element count MUST equal N, and its elements check against T in
  order. `[]` is the N = 0 case; a length mismatch is `E0815`. Without an
  expected Array type, `[1, 2, 3]` remains a `List<int>`.
- `Array::filled(value)` is the generic constructor: it creates N copies of
  `value` and is written with explicit type and const arguments, as in
  `Array::filled<int, 3>(0)` — there is no const inference. It is the only
  constructor besides the context-typed literal.
- `Array<T, N>` and `List<T>` are unrelated nominal types: there is no
  implicit conversion in either direction, and no FFI boundary for Array in
  this change.
