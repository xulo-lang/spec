# Add Fixed-Length Array

**Status:** draft · **Date:** 2026-10-27 · **Affects:**
`types/composite-types.md`, `types/type-relations.md`,
`expressions/literals.md`, `expressions/path-and-access.md`,
`expressions/control-flow.md`, `type-system/checking-rules.md`,
`type-system/errors.md`, `error-handling.md`, `memory-and-runtime.md`,
`builtins/prelude.md`, `types/README.md`, `README.md`

## Summary

This change builds on `add-const-generics` (const parameters and explicit
const arguments) and `change-collection-members` (the List member names and
semantics), and adds the fixed-length counterpart to `List<T>`: an **inline**
`Array<T, N>` whose length is part of the type.

> `struct Buffer<const N: int> { data: Array<int, N> }`
> `fn zeroes<const N: int>(): Array<int, N>`

Const arguments are the prerequisite's literals, module `const` names, and
const parameter names: `Array<int, 3>`, `Array<int, N>`, `zeroes<3>()`.
Version 1 adds no const arithmetic or inference (the prerequisite has none).

`Array<T, N>` is a value type with N inline elements, N ≥ 0: no `push`/`pop`/
`splice`/`resize`/`truncate`/`clear`, no length mutation. It keeps the read
and callback operations of List — `.len`, subscript, iteration, `slice`,
`indexOf`/`includes`/`join`, `entries`, `reverse`, `sort`, and
`forEach`/`map`/`filter`/`find`/… — with `map` returning `Array<U, N>`,
`filter`/`slice` returning `List`s, and `sort`/`reverse` mutating in place.
A literal is context-typed: `let a: Array<int, 3> = [1, 2, 3]`; a bare
`[1, 2, 3]` remains a `List`. `Array::filled(value)` constructs the N
elements of a generic array.

## Motivation

### List cannot serve as a fixed-size value

`List<T>` owns a growable backing store, has reference-like sharing, and
admits dynamic members. A fixed-size array is a different value: its length is
known statically, it can be laid out inline (like a tuple, C array, or Swift
`InlineArray`), it copies as a value, it cannot grow or shrink, and it is the
natural carrier for later foreign-layout work. Making List pretend to be both
would add length-changing operations to a type whose whole point is growth,
and would not give a type-level length.

### N must live in the type

`Array<T, N>` needs N in the type so that `Array<int, 3>` and `Array<int, 4>`
are different types, length checks are compile-time, and generic code can
abstract over N. The const-parameter mechanism (`add-const-generics`) supplies
that value parameter. A special array type with literal-only lengths could not
be used by `fn f<N>(a: Array<T, N>)`; it would be a family of concrete special
cases, not a fixed array in the generic language.

### Keep the List read surface, drop the write surface

The fixed array needs indexing, iteration, slicing, searching, joining,
callbacks, and sorting; it does not need growth. Reusing the List member names
and semantics (with `map` → `Array<U, N>`, `filter`/`slice` → `List`) keeps one
mental model and avoids a second set of names for the same operations. The
dynamic members — `push`/`pop`/`shift`/`unshift`/`splice`/`resize`/`truncate`/
`clear` — are absent, and any attempt to add one is a missing-member error.

### Dependencies

Array is the first consumer of the const-parameter mechanism and it shares
List's read/callback member names. Those are separate drafts,
`add-const-generics` and `change-collection-members`; the three are accepted in
that order. This change assumes their post-acceptance text and adds only the
Array type, its `N ≥ 0` domain, and its members. Folding it before either
prerequisite would leave undefined syntax or undefined member names.

## Scope

**In scope.** The inline value type `Array<T, N>`; its literal,
`Array::filled`, `.len`, subscript, iteration, read/callback/sort members, and
value-copy semantics; the non-negative `N` domain; the prelude/reserved name
`Array`; the new diagnostics.

**Out of scope.** The const-generic mechanism itself (see `add-const-generics`) and the List member changes (see `change-collection-members`);
length subtyping (N is an exact const argument, so different lengths are
unrelated types); `Array` ↔ `List` conversions; Array in FFI; Array as a
heap/reference type; a new literal syntax (the expected-type list literal is
reused); `Array` dynamic members.

## Compatibility

- No existing program changes meaning. The const-generic syntax is added by
  `add-const-generics`; this change adds only the `Array` type, its literal,
  and its members.
- `Array` joins the reserved type names; an explicit type-argument list that
  mixes types and const arguments is checked by the callee's parameter kinds.
- A bare `[1, 2, 3]` is still a `List<int>`; only an expected `Array<T, N>`
  makes it an array literal, and its element count must equal N.
- An `Array<T, N>` shares its elements by value: binding, assignment,
  argument passing, and return copy the N elements; it has no identity like a
  List's backing store. A const parameter is monomorphized, so it carries no
  runtime argument.
- `Array` and `List` are unrelated nominal types: no implicit conversion in
  either direction.
