# Checking Rules — Delta: Snake-Case Official Names

> **Delta** — replaces the `Γ ⊢ e.m : U` member-access bullet in
> “Identifiers and member access” in `type-system/checking-rules.md`.

- `Γ ⊢ e.m : U` when `e : T` and `T` (after alias expansion) declares member
  `m` of type `U`; unknown member → `E0102`, a non-`pub` member examined
  outside its module → `E0602`. On `e : unknown` there is no member at all,
  so `e.m` is `E0102`. The built-in collection and string types declare their
  inherent members with their types; a read-only property is not a place, and
  there is no map entry member form — `e.m` on a `Map` is a member lookup,
  never the entry `"m"`. On a `List<T>` or `Array<T, N>`,
  `index_of`/`last_index_of`/`includes` require `T` to admit `==`, and `join`
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
  succeeds only through a bound. A built-in static member
  (`Map::from_entries`, `Set::from_list`, `Array::filled`) is reached through
  `::`; it has no receiver.
