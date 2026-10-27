# Checking Rules — Delta: Array Typing

> **Delta** — assumes `change-collection-members` and `add-const-generics`;
> replaces the members/subscript rules with the Array-aware versions and adds
> the array-literal and `N >= 0` rules in
> `type-system/checking-rules.md`; the rest is unchanged.

- `Γ ⊢ e.m : U` when `e : T` and `T` (after alias expansion) declares member
  `m` of type `U`; unknown member → `E0102`, a non-`pub` member examined
  outside its module → `E0602`. The built-in collection and string types
  declare their inherent members with their types; a read-only property is not
  a place, and there is no map entry member form — `e.m` on a `Map` is a
  member lookup, never the entry `"m"`. On a `List<T>` or `Array<T, N>`,
  `indexOf`/`lastIndexOf`/`includes` require `T` to admit `==`, and `join`
  requires `T` to have a string form (`E0212`). The callback members check
  their function argument against `fn(T): unit`, `fn(T): U`,
  `fn(T): bool`, `fn(U, T): U`, or `fn(T, T): int`; `map` and `reduce`
  introduce a method type parameter inferred from the callback and initial
  value. The receiver is borrowed for the call, so a callback that mutably
  borrows the same collection is `E0404`/`E0405`. An `Array` has no dynamic
  members. Method resolution is the receiver type's built-in inherent member
  first, then inherent `impl` methods, then trait methods in scope, with
  explicit `Trait.method(recv)` always available; a method used without a call
  has its function type with the receiver bound. On a type parameter, access
  succeeds only through a bound. A built-in static member is reached through
  `::`; it has no receiver.

For `a[i]`: `a : List<T>` or `a : Array<T, N>` requires `i : int` and yields
`T`; any other type — `Map<K, V>` and `string` included — is `E0215`. The read
adds no optional: an out-of-range index is a runtime error. As an assignment
target the subscript is a place, so its base must be mutable and its value
assignable to the element type.

- An `Array<T, N>` type application requires the const argument N to be
  non-negative; a concrete negative N is `E0814`.
- A list literal checked against an expected `Array<T, N>` must have exactly N
  written elements and no spread; a mismatch is `E0815`. Its elements check
  against T in order, and `[]` is the N = 0 case.
