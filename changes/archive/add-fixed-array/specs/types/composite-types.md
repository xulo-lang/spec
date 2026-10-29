# Composite Types — Delta: Fixed-Length Array

> **Delta** — replaces the chapter introduction, the List-literal bullet, and
> the field-composition bullet, and adds `## Array<T, N>` after the List
> section of `types/composite-types.md`; the rest of the chapter is unchanged.
> This delta assumes the const-generic syntax from `add-const-generics` and the
> List member names from `change-collection-members`.

Replace the chapter introduction with:

Composite types are built from other types. They fall into three groups: the
built-in collections `List<T>`, `Array<T, N>`, `Map<K, V>`, and `Set<T>`; the
record types — nominal `struct` records and positional tuples; and the
constructors that combine types — `T?`, `T | U`, and `T & U`. This chapter
specifies the values each constructor admits, how values of those types are
written, and the rules by which values move between them.

Replace the first List bullet with:

- Absent an expected `Array` type, a list literal `[e1, e2, …]` has type
  `List<T>`, where `T` is the union of the element types: `[1, "two"]` is
  `List<int | string>`. Checked against an expected `Array<T, N>`, the same
  syntax is an array literal instead (see `## Array<T, N>` below).

Replace the field-composition bullet of `## struct` with:

- A struct's field types compose into the struct's value: the declaring type
  MUST NOT be reachable from its own fields through field positions, tuple
  elements, array element positions, and alias expansion without an intervening
  indirection — an indirection being `List<T>`, `Map<K, V>`, `Set<T>`, `T?`, a
  function type, or an enum type. `struct Node { node: Node }`, the mutual pair
  `struct A { b: B }` / `struct B { a: A }`, and `struct S { t: (S, int) }`
  are compile-time errors (`E0113`). An `Array<T, N>` is inline, so an array
  element position is part of that composition, not an indirection:
  `struct S { a: Array<S, 3> }` is `E0113`. A path through one of the listed
  indirections breaks the cycle: `struct Node { next: Node?, children:
  List<Node> }` is well-formed, and recursion through enum payloads is
  permitted in general (see [`enums.md`](enums.md)).

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
  - `.reverse` and `.sort` mutate the array; as in the List rule, the
    mutating members require the receiver to be a mutable place (`E0404` on
    an immutable binding); `sort` is stable and takes a comparator;
  - callbacks are synchronous, receive the element only, and cannot mutate
    the receiver (the ordinary borrow rules, `E0404`/`E0405`).
- There is **no** `.push`, `.pop`, `.shift`, `.unshift`, `.splice`, `.resize`,
  `.truncate`, or `.clear`: the length is N. A member outside this set is
  `E0102`.
- A list literal checked against an expected `Array<T, N>` is an array
  literal: its element count MUST equal N, and its elements check against T in
  order. `[]` is the N = 0 case; a length mismatch is `E0815`, and a literal
  that cannot be checked statically — a spread, or an abstract const parameter
  N — is `E0818`. Without an expected Array type, `[1, 2, 3]` remains a
  `List<int>`.
- `Array::filled<T, const N: int>(value: T): Array<T, N>` is the generic
  constructor: it creates N copies of `value` and is written with explicit type
  and const arguments, as in `Array::filled<int, 3>(0)` — there is no const
  inference.
  A call that omits the list is `E0816` and a partial list is `E0808` (the
  const-generic rules from `add-const-generics`). It is the only constructor
  besides the context-typed literal.
- `Array<T, N>` and `List<T>` are unrelated nominal types: there is no
  implicit conversion in either direction, and no FFI boundary for Array in
  this change.
- An `Array` has no string form: `str(a)` and `${a}` are `E0212`, as for any
  other collection operand of the intrinsics.
