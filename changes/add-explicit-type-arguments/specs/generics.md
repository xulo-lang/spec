# Generics — Delta: Explicit Type Arguments

> **Delta** — replaces the “Explicit type arguments” paragraph in `types/generics.md`.

A call MAY write the type arguments of a generic callee explicitly, in a list
between the callee and the argument list. The written arguments take the place
of inference for that call:

```xulo
let n = first<int>([1, 2, 3])      // T = int, written at the call
let s = first(["a", "b"])          // T = string, inferred as before
```

Rules:

- The list is written `<` … `>` immediately before the argument list,
  comma-separated with an optional trailing comma, and it is **all or
  nothing**: a call writes every type parameter of its callee or writes none.
- The number of type arguments MUST equal the number of type parameters the
  callee declares; a longer or shorter list is `E0808`.
- Each written type argument MUST satisfy every bound of its parameter,
  exactly as an inferred one must; a violation is `E0801`.
- When the list is written, inference is suppressed for those parameters: no
  type variable is introduced for them, no unification constraint is raised
  against the value arguments, and an expected type MUST NOT override a
  written argument. The value arguments are then checked against the
  parameter types with the arguments substituted, and a mismatch is the
  ordinary argument error `E0202`:

  ```xulo
  let a = first<int>([1, 2, 3])      // OK: `list<int>` for `list<T>`
  let b = first<int>(["a", "b"])     // error[E0202]: expected `list<int>`, found `list<string>`
  ```

- The type of the call is the declared return type with the written
  arguments substituted, and an expected type still checks that result:

  ```xulo
  fn empty<T>(): list<T> { [] }

  let xs: list<int> = empty<int>()      // OK
  let ys: string = empty<int>()         // error[E0201]: expected `string`, found `list<int>`
  ```

- A list written before a callee that declares no type parameters is `E0807`.
- The list is available wherever a call is written — on a function, a method,
  a trait static, or a namespace member — whenever the callee declares type
  parameters:

  ```xulo
  let a = describe<Rectangle>(r)      // T = Rectangle, checked against `T: Area`
  let b = Task.all<int>([t1, t2])     // T = int for the built-in namespace member
  ```

- `>>` is a single token, so the nested-bracket spacing rule of this chapter
  applies to a written list as well: `first<Map<string, int> >(m)` is
  well-formed and `first<Map<string, int>>(m)` is not.

Whether a `<` after a callee begins such a list is decided by the
disambiguation rule of [design.md](../design.md), and the production itself is
given in the grammar delta there; this chapter is concerned only with what a
written list means once it is parsed.

**Omission is unchanged.** A call that writes no type arguments is checked
exactly as specified before this change: type arguments are determined by
unification between the declared parameter types and the argument types, an
expected type MAY determine what the arguments leave undetermined, and a
parameter that remains under-constrained MUST be reported with a diagnostic.
The written list is an additional way to supply what inference would
otherwise have to find; it adds no new freedom to a call that omits it.
