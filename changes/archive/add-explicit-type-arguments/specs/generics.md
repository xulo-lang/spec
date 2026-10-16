# Generics — Delta: Explicit Type Arguments

> **Delta** — replaces the **Inference** section of `types/generics.md`.

## Inference

A call MAY write the type arguments of a generic callee explicitly, in a list
between the callee and the argument list. The written arguments take the place
of inference for that call:

```xulo
let n = first<Int>([1, 2, 3])      // T = Int, written at the call
let s = first(["a", "b"])          // T = String, inferred as before
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
  let a = first<Int>([1, 2, 3])      // OK: `List<Int>` for `List<T>`
  let b = first<Int>(["a", "b"])     // error[E0202]: expected `List<Int>`, found `List<String>`
  ```

- The type of the call is the declared return type with the written
  arguments substituted, and an expected type still checks that result:

  ```xulo
  let xs: Int = first<Int>([1, 2, 3])      // OK: the call has type Int
  let ys: String = first<Int>([1, 2, 3])   // error[E0201]: expected `String`, found `Int`
  ```

- A list written before a callee that declares no type parameters is `E0807`.
- The list is available wherever a call is written — on a function, a method,
  a trait static, or a namespace member — whenever the callee declares type
  parameters:

  ```xulo
  let a = describe<Rectangle>(r)      // T = Rectangle, checked against `T: Area`
  let b = Task.all<Int>([t1, t2])     // T = Int for the built-in namespace member
  ```

- `>>` is a single token, so the nested-bracket spacing rule of this chapter
  applies to a written list as well: `first<Map<String, Int> >(m)` is
  well-formed and `first<Map<String, Int>>(m)` is not.

Whether a `<` after a callee begins such a list is decided by the
disambiguation rule of
[`../expressions/calls.md`](../expressions/calls.md), and the production itself
is in [`../grammar.md`](../grammar.md); this chapter is concerned only with
what a written list means once it is parsed.

**Inference for an omitted list is unchanged.** A call that writes no type
arguments is checked exactly as follows.

- Type arguments are determined by *unification* between the declared parameter
  types and the argument types. Unification decomposes type constructors
  structurally:

  | Constraint | Result |
  |------------|--------|
  | `List<T>` ≟ `List<Int>` | `T = Int` |
  | `Map<K, V>` ≟ `Map<String, Int>` | `K = String`, `V = Int` |
  | `T` ≟ `fn(Int): Boolean` | `T = fn(Int): Boolean` |
  | `T?` ≟ `String?` | `T = String` |
  | `List<T>` ≟ `Map<String, Int>` | fails (different heads) |

- A parameter occurring twice in a signature receives one solution; if two
  constraints demand different types for it, inference fails and the later
  argument is reported as a mismatch against the type the earlier one fixed
  (`E0202`):

  ```xulo
  fn both<T>(a: List<T>, b: List<T>): T { a[0] }
  let t = both([1], ["x"])    // error[E0202]: T is Int from `a`; `List<String>` does not match `List<Int>`
  ```
- An *expected type* MAY determine what the arguments leave undetermined — an
  empty list literal, a numeric literal, a closure with unannotated parameters.
  It MUST NOT override a type already fixed by the arguments: in
  `let bad: String = first([1, 2])`, `T` is `Int`, so the declaration is an
  error rather than an invitation to choose `T = String`. When the list is
  written, the expected type checks the substituted result instead of feeding
  anything back into the arguments.
- Inference MUST fail with a diagnostic when a parameter remains
  under-constrained:

  ```xulo
  fn empty<T>(): List<T> { [] }
  let xs = empty()             // error: T is not determined
  let ys: List<Int> = empty()  // OK: T = Int from the expected type
  ```

  Such a call is repaired by writing the list — `empty<Int>()` — or by
  annotating its context. Constraint generation and solving are specified in
  [`../type-system/checking-rules.md`](../type-system/checking-rules.md), and
  the diagnostics raised on failure are listed in
  [`../type-system/errors.md`](../type-system/errors.md).

The written list is an additional way to supply what inference would
otherwise have to find; it adds no new freedom to a call that omits it.
