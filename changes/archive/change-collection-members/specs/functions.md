# Functions — Delta: Built-in Members and Method Resolution

> **Delta** — replaces the first three method bullets of the “Methods and
> `self`” section of `functions.md` and adds the static-members bullet at the
> end of the bullet list; bullets 4–6, the surrounding prose, and the
> `Counter` example are unchanged.

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
- Inside a body bounded by `T: Trait`, both `x.method(args)` and
  `Trait.method(x)` are permitted.
- A method call whose receiver may be `null` is written with optional
  chaining, `receiver?.method()`; the expression evaluates to `null` instead
  of evaluating the call
  ([`expressions/path-and-access.md`](expressions/path-and-access.md)).
- The receiver is the first parameter of the method's function type; using a
  method as a value with the receiver bound is specified in
  [`expressions/calls.md`](expressions/calls.md).
- **Static members.** `Map::fromEntries(entries)` and
  `Set::fromList(values)` are associated members of the built-in types,
  reached through `::` like the associated names of an `impl` block. They have
  no receiver and are never selected by `.`.
