# Checking Rules — Delta: Const Generic Arguments

> **Delta** — replaces the value-binding bullet, the bounds bullet, the
> “Explicit type arguments” block, and the “Type applications” bullet in
> `type-system/checking-rules.md`; adds the const-argument rule to both call
> and type-application checking. The rest is unchanged.

Replace the value-binding bullet with:

- **Value bindings** — name to type and mutability: `let`/`let mut` bindings,
  `const`s, generic const parameters, parameters, loop variables, pattern
  bindings, component state declarations, and `self`.

Replace the bounds bullet with:

- **Bounds.** Every type argument — inferred at a call or written in a type
  application — MUST satisfy its parameter's bounds (`E0801`); a bound that is
  not a visible `trait` is `E0806`, and a bound that applies type arguments to
  a trait is `E0811`. A const argument has no bound. A `trait` declaration
  that declares a generic parameter list is `E0810`
  ([`../types/generics.md`](../types/generics.md), [`../types/traits.md`](../types/traits.md)).

Replace the “Explicit type arguments” block with:

- **Explicit generic arguments.** A call MAY write its callee's generic
  arguments, in which case they are checked in this order, before any
  constraint exists for the call:

  - **Arity.** The list length MUST equal the number of generic parameters the
    callee declares, type and const combined; a longer or shorter list is
    `E0808`. Every parameter is written or none is — a partial list is the
    same failure, not a hybrid form that leaves the remainder to inference.
  - **Kinds.** At a type parameter position the argument MUST be a type; at a
    const parameter position it MUST be an integer literal, a `const` binding
    (module-level or block-level), or a const parameter name, evaluated to a
    compile-time `int`. A mismatch is `E0813`; a const parameter whose
    declared type is not `int` is `E0812`.
  - **Bounds.** Each written type argument MUST satisfy every bound of its
    parameter, reported as `E0801`, exactly as an inferred argument must be;
    a bound that is not a visible `trait` is `E0806`. A written const
    argument has no bound.
  - **Substitution without inference.** For each type parameter the binding
    `Tᵢ := Aᵢ` is made before any constraint is generated: no type variable is
    introduced for it, no unification constraint is raised against the value
    arguments, and no expected type may override it. A written const argument
    is used as given; no const variable is introduced. The value arguments
    are then checked against the substituted parameter types, and a mismatch
    is the ordinary argument error (`E0202`).
  - **Result.** The type of the call is the declared return type with every
    `Tᵢ := Aᵢ` substituted and every const parameter replaced by its value; an
    expected type checks that result and feeds nothing back into the written
    arguments.

  A call to a callee that declares a const parameter and writes no list is
  `E0816`; const arguments are never inferred. A list written before a callee
  that declares no generic parameters is `E0807`, including a value of
  function type (`f<int>(…)` where `f: fn(int): int`) and a `struct` or `enum`
  constructor. The rule applies wherever the callee ends in an identifier — a
  method (`r.first<int>(xs)`), a trait static (`Trait.method<int>(x)`), a
  namespace member (`Task.all<int>(…)`), or a component
  (`Counter<int>() { … }`).

Replace the type-application bullet with:

- **Type applications.** Every type application MUST be well-formed: the
  argument count equals the declaration's generic-parameter count, type and
  const combined; a generic name never appears without its arguments; and a
  type or const parameter is never applied — all `E0230`
  ([`../types/generics.md`](../types/generics.md)). A const argument in a
  type application is checked by kind exactly as at a call; a mismatch is
  `E0813`.

In the `impl completeness` bullet, replace “its parameters instantiate to
the type and its bounds are satisfied” with “its type and const parameters
instantiate to match the type and its bounds are satisfied”.
