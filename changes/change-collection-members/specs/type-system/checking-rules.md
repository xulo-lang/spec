# Checking Rules — Delta: Built-in Members and List-Only Subscripts

> **Delta** — replaces the “Identifiers and member access” and “Enum variant
> paths and subscript” bullets of `type-system/checking-rules.md`; the rest of
> the chapter is unchanged.

- `Γ ⊢ x : T` when `x` resolves to a binding, declaration, or type of type `T`
  in the resolution order of [`../names.md`](../names.md); an identifier that
  resolves to nothing — or a type name written where a value is required — is
  `E0101`.
- `Γ ⊢ e.m : U` when `e : T` and `T` (after alias expansion) declares member
  `m` of type `U`; unknown member → `E0102`, a non-`pub` member examined
  outside its module → `E0602`. The built-in collection and string types
  declare their inherent members with their types; a read-only property is not
  a place, and there is no map entry member form — `e.m` on a `Map` is a
  member lookup, never the entry `"m"`. On a `List<T>`,
  `indexOf`/`lastIndexOf`/`includes` require `T` to admit `==`, and `join`
  requires `T` to have a string form (`E0212`). The callback members check
  their function argument against `fn(T): unit`, `fn(T): U`,
  `fn(T): bool`, `fn(U, T): U`, or `fn(T, T): int`; `map` and `reduce`
  introduce a method type parameter inferred from the callback and initial
  value. The receiver is borrowed for the call, so a callback that mutably
  borrows the same list is `E0404`/`E0405`. Method resolution is the
  receiver type's built-in inherent member first, then inherent `impl`
  methods, then trait methods in scope, with explicit `Trait.method(recv)`
  always available;
  a method used without a call has its function type with the receiver bound.
  On a type parameter, access succeeds only through a bound. A built-in static
  member (`Map::fromEntries`, `Set::fromList`) is reached through `::`; it has
  no receiver.
- `Γ ⊢ p.i : Tᵢ` when `p : (T₁, …, Tₙ)` and `i` is a decimal literal naming
  an existing position — `0 ≤ i < n`; a positional access on a non-tuple
  type or at or past the arity → `E0220`
  ([`../expressions/path-and-access.md`](../expressions/path-and-access.md)).

### Enum variant paths and subscript

`E::V` requires `E` to resolve to an `enum` and `V` to be one of its variants
(`E0102` otherwise); a payload-less variant has type `E`, and a variant with a
payload is a constructor whose call must supply exactly the declared payload
arguments (`E0202`, `E0207`, `E0208`). `::` is used only for variants, in
expressions and patterns alike.

For `a[i]`: `a : List<T>` requires `i : int` and yields `T`; any other type —
`Map<K, V>` and `string` included — is `E0215`. The read adds no optional: an
out-of-range index is a runtime error. As an assignment target the subscript
is a place, so its base must be mutable and its value assignable to the element
type.
