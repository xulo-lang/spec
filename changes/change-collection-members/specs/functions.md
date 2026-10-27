# Functions — Delta: Built-in Members and Method Resolution

> **Delta** — replaces the method bullets in the “Methods” section of
> `functions.md` and adds the static-members bullet; the surrounding prose
> and the `Counter` example are unchanged.

- Methods are functions declared inside an `impl` block. The built-in
  collection and string types also declare **inherent members** specified
  with their types
  ([`types/composite-types.md`](types/composite-types.md),
  [`types/primitive-types.md`](types/primitive-types.md)); the methods among
  them are methods in the call and method-value sense, while `.len`, `.size`,
  and `.byteLen` are read-only properties. None is an `impl` declaration, and
  a program cannot add an inherent member to them; `impl Trait for` a built-in
  type is an ordinary trait implementation. The first parameter is
  the **receiver**, written `self` (an immutable borrow) or `mut self` (a
  mutable borrow). `self` is an explicit parameter, never implicit. A method
  that assigns to `self` or a field of `self` MUST declare `mut self`.
- At the call site the receiver is not written as an argument:
  `c.increment()` supplies `c` to the declared `self` parameter.
- **Method resolution order:** `value.method(…)` resolves first to the
  receiver type's built-in inherent member, then to an inherent method
  declared in an `impl` block for the receiver's type, and otherwise to a
  trait method provided by an implementation in scope at the call site.
  Exactly one candidate MUST result; ambiguity is an error (`E0809`) and MUST
  be disambiguated with explicit dispatch `Trait.method(receiver)`. Explicit
  dispatch is always available.
- **Static members.** `Map::fromEntries(entries)` and
  `Set::fromList(values)` are associated members of the built-in types,
  reached through `::` like the associated names of an `impl` block. They have
  no receiver and are never selected by `.`.
