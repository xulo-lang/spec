# Design: inline fixed arrays

## Prerequisites

This design builds on two drafts that must be accepted first:

- `add-const-generics` supplies `<const N: int>`, explicit const arguments
  (integer literals, module `const` names, const parameter names), no const
  arithmetic or inference, and monomorphization over const values.
- `change-collection-members` supplies the List member names and semantics
  (`.len`, `.slice`, `.indexOf`, `.join`, `.entries`, the callback members,
  and `sort`) that Array shares.

Array is then the first consumer of const generics; this change adds only the
`Array` type, its `N ≥ 0` domain, and its members.

### The Array length domain

`Array<T, N>` requires `N >= 0`. A concrete negative const argument (for
example a module `const M = -1` used as `Array<int, M>`) is `E0814`. A const
*parameter* is abstract inside the generic declaration, so `Array<T, N>` with
a const parameter N is allowed; each instantiation checks N against the domain
and an instantiation with a negative value is `E0814`.

## Array<T, N>

`Array<T, N>` is a built-in **inline value type** with exactly N elements:

| Member | Result |
|--------|--------|
| `.len` | `int`, the constant N; read-only, O(1) |
| `a[i]` / `a[i] = v` | the element `T`; index `int` in `0 … len − 1`, otherwise a runtime failure |
| `for x in a` | elements in index order |
| `.slice(start, end?)` | `List<T>`; a runtime-length copy |
| `.indexOf` / `.lastIndexOf` / `.includes` | `int` / `int` / `bool`; `==`, NaN not found |
| `.join(separator = ",")` | `string`; the `str` element domain (`E0212`) |
| `.entries()` | `List<(int, T)>`, in order |
| `.reverse()` / `.sort(comparator)` | `unit`; mutating; sort is stable |
| `.forEach` / `.map` / `.filter` / `.find` / `.findIndex` / `.some` / `.every` / `.reduce` | as for List, except `.map` returns `Array<U, N>`; `.filter` returns `List<T>` |

- There is **no** `push`, `pop`, `shift`, `unshift`, `splice`, `resize`,
  `truncate`, or `clear`: the length is N. A member outside this set is
  `E0102`.
- `.map` preserves the length, so its result is `Array<U, N>`; `.filter` and
  `.slice` have runtime lengths, so they return `List`s. Callbacks follow the
  List rules: synchronous, element only, receiver borrowed, a callback panic
  stops the call.
- The read/callback/sort members otherwise share List's semantics and
  diagnostics; Array has no dynamic members to shadow them.
- `a[i]` and iteration are core syntax; a subscript outside `0 … N − 1` is a
  runtime failure, joining the closed tables as *Array index out of bounds*.
- Values are inline and have value semantics: binding, assignment, argument
  passing, and return copy the N elements; there is no shared backing store,
  and no identity relation depends on storage. `Array<T, N>` is invariant in
  `T` and exact in N; there is no subtyping between different N and no
  conversion to or from `List`.

### Literal and constructor

- A list literal checked against an expected `Array<T, N>` is an **array
  literal**: its element count MUST equal the constant N, and its elements
  check against T in order. `let a: Array<int, 3> = [1, 2, 3]` is the form;
  `[]` is the N = 0 case. A literal whose length is not equal to N is
  `E0815`.
- Without an expected Array type, `[1, 2, 3]` remains a `List<int>`; there is
  no bare array literal and no new literal syntax.
- In a generic declaration N is abstract, so a literal cannot prove its length
  equals N. `Array::filled(value)` is the generic constructor: it creates N
  copies of `value` and is written with explicit type and const arguments,
  e.g. `Array::filled<int, 3>(0)`; const inference is not available. It is the
  only constructor besides the context-typed literal.

### No FFI, no conversions

An inline Array is not yet an FFI boundary type (the FFI chapter admits only
scalars and opaque handles), and there is no `Array` ↔ `List` conversion. Both
are deliberate follow-ups; the fixed array is a language value first.

## Alternatives considered

### A special literal-only Array without const generics

**Cost.** `Array<int, 3>` would be a built-in special case; `fn f(a:
Array<int, N>)` could not abstract over N, and a library could not be generic
over array length. Rejected: the point of a type-level N is generic code.

### Tuple types as fixed sequences

**Cost.** `(T, T, T)` supplies arity but no `Array<T, N>` type, no uniform
indexing/iteration, and no length abstraction. Rejected.

### A heap/reference fixed array

**Cost.** A heap object with a fixed length is List without growth; it loses
the inline layout and value-copy motivation, and still needs const generics for
N. Rejected for this change; a reference fixed array is a different design.

## Consequences

### Type chapters and prelude

- `Array` is a built-in type name; the prelude's type table and reserved-name
  list gain it. Its members are specified in the type chapter; the literal,
  subscript, iteration, and error rows are added to their chapters.
- No grammar change: `Array<T, N>` uses the const-generic syntax supplied by
  `add-const-generics`; the array literal reuses the list-literal syntax.

### Typing and diagnostics

- A const argument outside `Array`'s `N >= 0` domain is `E0814`; an array
  literal whose element count differs from N is `E0815`.
- `a[i]` out of range is a runtime failure; `Array::filled`/literal/member
  arguments use the ordinary call and checking rules.

### Memory and monomorphization

- `Array<T, N>` is inline, with value copy; N is substituted at instantiation,
  so no ordinary operation observes a const argument at runtime.
- Monomorphization means a large N can increase code size; the specification
  gives no size promise beyond the as-if rule.

### Documentation

- New Array type/literal/member text; subscript/iteration/error-table
  additions; examples in the prelude and type chapters; no FFI or conversion
  promise. The const-generic prose lives in `add-const-generics`.
