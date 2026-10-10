# Checking Rules — Delta: Explicit Type Arguments

> **Delta** — in `type-system/checking-rules.md`, adds the **Explicit type arguments** bullet after the **Bounds** bullet of “Declaration rules”, and replaces step 4 of “Constraint generation and solving”.

## Declaration rules

- **Explicit type arguments.** A call MAY write its callee's type arguments,
  in which case they are checked in this order, before any constraint exists
  for the call:

  - **Arity.** The list length MUST equal the number of type parameters the
    callee declares; a longer or shorter list is `E0808`. Every parameter is
    written or none is — a partial list is the same failure, not a hybrid
    form that leaves the remainder to inference.
  - **Bounds.** Each written argument MUST satisfy every bound of its
    parameter, reported as `E0801`, exactly as an inferred argument must be;
    a bound that is not a visible `trait` is `E0806`.
  - **Substitution without inference.** For each parameter the binding
    `Tᵢ := Aᵢ` is made before any constraint is generated: no type variable is
    introduced for it, no unification constraint is raised against the value
    arguments, and no expected type may override it. The value arguments are
    then checked against the substituted parameter types, and a mismatch is
    the ordinary argument error (`E0202`).
  - **Result.** The type of the call is the declared return type with every
    `Tᵢ := Aᵢ` substituted; an expected type checks that result and feeds
    nothing back into the written arguments.

  A list written before a callee that declares no type parameters is `E0807`,
  including a value of function type (`f<int>(…)` where `f: fn(int): int`) and
  a `struct` or `enum` constructor. The rule applies wherever the callee ends
  in an identifier — a method (`r.first<int>(xs)`), a trait static
  (`Trait.method<int>(x)`), a namespace member (`Task.all<int>(…)`), or a
  component (`Counter<int>() { … }`).

## Constraint generation and solving

4. **Bidirectional propagation.** An expected type is pushed into literals,
   empty collections, closures, and generic calls before they are inferred
   bottom-up; it never overrides a type the arguments have already fixed, and
   it never overrides a type argument written at the call. A call that writes
   its list has a closed type as soon as the list is written, so the expected
   type checks that type rather than feeding anything back into the arguments.
