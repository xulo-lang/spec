# Type Relations and Inference

This chapter defines the relations that hold *between* types — equality,
subtyping, assignability, and coercion — together with the variance of the
built-in type constructors and the local inference procedure that produces
types inside function bodies. The rules that consume these relations are in
[`../type-system/checking-rules.md`](../type-system/checking-rules.md).

## Type Equality

Two types are **equal** when, after expanding every type alias, they have
identical structure. Equality is written `A = B` and is an equivalence relation.

- **Alias transparency.** A `type` alias introduces no new type: `type Name = X`
  makes `Name` and `X` equal types, everywhere and for every purpose. Aliases
  may be expanded to any depth, and expansion terminates because an alias MUST
  NOT expand to itself.
- **Definitional equalities.** The following pairs are equal by definition:
  `T?` and `T | Null`; `Number` and `Int | Float`; a function type written
  without a return type and the same type written `fn(…): Unit`.
- **Generic applications.** `C<A₁, …, Aₙ>` and `C<B₁, …, Bₙ>` are equal exactly
  when the heads `C` are the same declaration and `Aᵢ = Bᵢ` for every `i`.
  Different arguments give different types: `List<Int>` and `List<String>` are
  unrelated, as are `Pair<Int, Int>` and `Pair<Int, String>`.
- **Nominal types.** A `struct`, `enum`, or `trait` is equal only to the type
  introduced by its own declaration (and to aliases of it). Two declarations with
  identical members are still distinct types.
- **Tuples.** `(T₁, …, Tₙ) = (U₁, …, Uₘ)` exactly when `n = m` and
  `Tᵢ = Uᵢ` for every position `i`: a tuple's arity is part of its identity,
  so two tuples of different lengths are never equal types.

Type equality must not be confused with `==`, which compares *values*; see
[`../expressions/operators.md`](../expressions/operators.md).

## Subtyping

Subtyping, written `A <: B`, holds when every value of `A` is a value of `B`
without any conversion. It is reflexive (`T <: T`) and transitive, and it is the
least relation containing the following rules.

| Rule | Notes |
|------|-------|
| `Null <: T?` | the null value inhabits every optional |
| `T <: T?` | every value is a value of its optional |
| `A <: A \| B`, `A <: B \| A` | union injection |
| `A & B <: A`, `A & B <: U` for `B <: U` | intersection projection, **not** the converse |
| `"s" <: String` | a string literal type is a string |
| `"s" <: "s" \| "t"` | literal union membership |
| `Int <: Number`, `Float <: Number` | from `Number = Int \| Float` |
| `A <: B` implies `Task<A> <: Task<B>` | `Task<T>` is covariant |
| `A <: B` implies `A? <: B?` | the optional is covariant |
| `T <: Unknown` | `Unknown` is the top type: every type inhabits it |

Rules by type family:

- **Optionals and null.** `T <: T?` and `Null <: T?` hold; neither `Null <: T`
  nor `T <: Null` holds for a non-optional `T`. `T?` is exactly `T | Null`, so
  optional subtyping is an instance of the union rule.
- **Unions and intersections.** A member is a subtype of the union containing
  it, in either position, and a union is a subtype of a union that contains all
  of its members. An intersection is a subtype of each of its operands:
  `T & U <: T` and `T & U <: U`. The converses do **not** hold — a `T` is not a
  subtype of `T & U`, and neither `T` nor `U` alone is a subtype of the other.
- **Numeric types.** `Int` and `Float` are subtypes of `Number` only because
  `Number` is defined as `Int | Float`. No other numeric relation holds:
  `Int <: Float` is false, `I32 <: I64` is false, and no fixed-bit type is a
  subtype of another. Literal adaptation between numerics is coercion, not
  subtyping (see [`../type-system/coercion.md`](../type-system/coercion.md)).
- **The top type.** `T <: Unknown` holds for every `T`, and `Unknown <: T`
  holds only when `T` is `Unknown`. No other rule relates `Unknown` to another
  type, so nothing narrows by itself: a value of type `Unknown` is used only
  through a `match` type pattern or the `is` operator
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)).
- **Nominal types.** `struct` and `enum` types are nominal. A `struct` with more
  fields is **not** a subtype of one with fewer; field count never creates
  subtyping between distinct declarations. Two different enums are never in a
  subtype relation.
- **Tuples.** Arity is part of a tuple's identity and is never a source of
  subtyping: `(T₁, …, Tₙ) <: (U₁, …, Uₘ)` holds only when `n = m` and
  `Tᵢ = Uᵢ` for every `i` — that is, when the two types are equal. There is
  no width relation (a shorter tuple is not "missing fields" of a longer
  one), no depth relation, and no common supertype across arities.
- **Function types.** Parameters are contravariant and the return type is
  covariant: `fn(A₁, …, Aₙ): R₁ <: fn(B₁, …, Bₙ): R₂` exactly when the arities
  are equal, `Bᵢ <: Aᵢ` for every `i`, and `R₁ <: R₂`. Nothing else relates
  function types; in particular, differing arity is never a subtype relation.
- **Bounds are not supertypes.** A bound `T: Area` constrains which types may be
  substituted for `T`; it does not make `T` a subtype of `Area`. See
  [`traits.md`](traits.md).
- **Built-in generics.** Whether subtyping lifts through `List<T>`, `Map<K, V>`,
  `Set<T>`, `struct`s, `enum`s, and tuples is decided by their variance,
  below.

## Assignability vs subtyping vs coercion

Three distinct relations are used by this specification.

| Relation | Kind | Used where |
|----------|------|------------|
| Subtyping `A <: B` | a relation on types alone; no conversion | function types, unions, variance, bound reasoning |
| Coercion `A ⇝ B` | a set of directed implicit conversions | adapting a value where subtyping does not hold, e.g. a numeric literal to a fixed-bit type |
| Assignability `A ≼ B` | the judgement applied at checking sites | `let` annotations, arguments, `return`, assignment targets, list, map, and tuple literals, `match` arm results, expected types |

- **Assignability** is the practical question "may a value of type `A` be used
  where `B` is expected?". `A ≼ B` holds exactly when `A <: B`, or when a
  coercion `A ⇝ B` applies. It is therefore defined in terms of the other two,
  but it is a different notion: subtyping never converts anything, and a
  coercion may exist where no subtype relation holds.
- **Subtyping** is context-free: it depends only on the two types.
  **Assignability** is context-sensitive, because whether a coercion applies can
  depend on the expected type (a literal is adapted differently toward `Int`,
  `U32`, and `Float`).
- **Coercion** is owned by
  [`../type-system/coercion.md`](../type-system/coercion.md), which lists every
  permitted conversion, including numeric literal adaptation, the `Int + Float`
  promotion of the arithmetic operators, and the string coercion used by
  template interpolation.

## Variance

Variance describes how a type constructor's subtype relation depends on its
arguments: `C<T>` is **covariant** in `T` when `A <: B` implies `C<A> <: C<B>`;
**contravariant** when it implies `C<B> <: C<A>`; **invariant** when neither
lifts except when `A = B`.

| Constructor | Variance | Reason |
|-------------|----------|--------|
| `List<T>` | invariant in `T` | elements may be replaced through a `List` value |
| `Map<K, V>` | invariant in `K` and `V` | entries may be inserted, replaced, and removed |
| `Set<T>` | invariant in `T` | membership may be changed |
| `(T₁, …, Tₙ)` | invariant in each element | elements may be written through a mutable tuple |
| `T?` | covariant | an optional only produces a `T` or `null` |
| `T \| U` | covariant in each operand | a union only produces one of its members |
| `T & U` | covariant in each operand | an intersection produces both operands |
| `fn(A): R` | contravariant in `A`, covariant in `R` | it consumes `A` and produces `R` |
| `Task<T>` | covariant | `await` only produces a `T` |
| `Range<T>` | covariant | iteration only reads the bounds |
| `struct S<T>` / `enum E<T>` | invariant in `T` | nominal, and fields may be written |
| `Result<T, E>` | invariant in `T` and `E` | a built-in with fixed `Ok` and `Err` payloads |
| `type X<T> = …` | follows its expansion | aliases are transparent |

Consequences:

- Because the collections are invariant, `List<Int>` and `List<Number>` are
  unrelated types: neither is assignable to the other. This is what makes
  inference precise — unifying `List<T>` with `List<Int>` has the single
  solution `T = Int`.
- Invariance constrains *existing values*, not construction. A list literal
  checked against the expected type `List<Number>` types its elements as
  `Number`, and `let xs: List<Number> = [1, 2, 3]` is well-formed.
- Function variance is why a function that accepts *any* `Number` may be used
  where a function accepting only `Int` is expected, but not the other way
  around.

## Local Inference

Xulo uses **local inference**: types are inferred inside function bodies, and
never across function signatures.

- **Signatures are explicit.** Every parameter of a module-level `fn`
  declaration MUST be annotated with its type. A function declared `pub` MUST
  declare its return type. A return type that is omitted means `Unit`, and the
  return type of a function is never inferred from its body. Closures MAY omit
  parameter and return annotations when the expected type or the body
  determines them; an unannotated parameter that stays undetermined is an
  error.
- **No cross-function inference.** A caller never learns anything from the body
  of a callee: only the declared signature participates in checking, so a
  module's exported declarations are sufficient to check its clients without
  their bodies (see [`../modules/README.md`](../modules/README.md)).
- **Literal typing.** An integer literal is `Int` and a float literal is `Float`
  unless the expected type adapts it: a literal checked against `U32`, `F32`,
  `Number`, or another numeric type takes that type when it is in range. A
  string literal is `String`, except when the expected type is a union of
  string literal types, in which case it takes the matching literal type, so
  `type Status = "active" | "inactive"` accepts `"active"`. No literal is ever
  typed `Unknown`: `Unknown` is written, never inferred.
- **Expected-type propagation.** Checking is bidirectional. When the context
  provides an expected type, it is pushed into literals, list and map
  literals, empty collections, closures, and generic calls; otherwise the
  expression is checked bottom-up and the result is then checked against the
  context. An expected type never overrides a type that inference has already
  fixed from other evidence.
- **Annotations are constraints.** `let x: T = e` requires `e ≼ T`; it does not
  rewrite the type of `e`.

```xulo
let a = 42                 // Int
let b = 3.14               // Float
let c: Number = 42         // int literal adapted to Number
let d: U32 = 800           // int literal coerced to U32
let e: Status = "active"   // the literal checks as "active"
let f = null               // error: no expected type, nothing to infer
```

## Constraint Solving (informal)

Checking a body generates constraints that are then solved.

1. **Generation.** Constraints come from every site where a type must be
   related: a `let` annotation, an argument against a parameter type, a
   `return` expression against the declared return type, an assignment target,
   the elements of a literal against the expected type, a `match` scrutinee
   against its patterns, the result types of `match` and `if` arms against each
   other, and a generic argument against the type parameter it instantiates.
2. **Unification.** Constraints between types containing type variables are
   solved by unification: equal constructors decompose into their arguments, a
   free variable binds to the type on the other side, and a variable already
   bound must unify with the new type or the solve fails. Because the
   constructors in the table above are invariant where they are writable,
   decomposition has at most one solution.
3. **Occurs check.** If solving would require a variable to be a proper part of
   itself — `T = List<T>` — the constraint set is infinite and MUST be rejected
   as a type error rather than accepted as a cyclic type.
4. **Assignability.** Once variables are solved, the remaining constraints are
   discharged as assignability checks between closed types, using subtyping and
   coercion as defined above.

A constraint set that cannot be solved, that stays under-constrained, or that
fails the occurs check MUST be reported as a compile-time diagnostic; see
[`../type-system/errors.md`](../type-system/errors.md) for the diagnostics and
[`../type-system/checking-rules.md`](../type-system/checking-rules.md) for the
complete rules.

## Type Errors vs Coercions

When a value of type `A` is required at a position of type `B`, exactly three
outcomes are possible: `A ≼ B` holds because `A <: B`, and nothing further
happens; `A ≼ B` holds only because a coercion applies, and that conversion is
part of the meaning of the program; or neither holds, and the program is
rejected with a type error naming both types. There is no fallback in which
unrelated types are accepted, and there is no explicit conversion between
unrelated types other than the operations the language defines — a value is
converted by an intrinsic such as `str`, by a `match` that produces the target
type, or not at all. Coercions that are not listed in
[`../type-system/coercion.md`](../type-system/coercion.md) never apply, so a
failed assignability check is always a genuine type error.
