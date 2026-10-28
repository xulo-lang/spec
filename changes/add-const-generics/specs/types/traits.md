# Traits — Delta: Const Parameters and Impl Matching

> **Delta** — replaces the trait-parameter bullet, the generic-impl bullet,
> the “When an impl applies” bullet, and the `#Derive` generic-type bullet in
> `types/traits.md`; the rest is unchanged.

Replace the trait-parameter bullet with:

- A trait declares **no generic parameters**. Its grammar accepts a generic
  parameter list only so that the attempt can be diagnosed; declaring one —
  `trait Convert<T> { … }` or `trait Sized<const N: int> { … }` — is a
  compile-time error (`E0810`), and a trait's methods therefore never mention
  type or const parameters. A trait is a fixed contract: a type that should
  convert to several targets implements one trait per target, not one
  parameterized trait ([`generics.md`](generics.md)).

Replace the generic-impl bullet with:

- An `impl` block MAY be generic, with its own type and const parameter list
  and bounds on type parameters, and MAY declare inherent methods as well as
  trait methods:

  ```xulo
  struct Pair<A, B> { first: A, second: B }
  struct Tagged<const N: int, T> { value: T }

  impl<T> Pair<T, T> {
    fn duplicated(self): T { self.first }
  }

  impl<const N: int, T> Tagged<N, T> {
    fn size(self): int { N }
  }
  ```

Replace the “When an impl applies” bullet with:

- **When an impl applies.** A generic `impl` applies to a type exactly when
  its type and const parameters can be instantiated so that the implemented
  type — `Pair<T, T>` or `Tagged<N, T>` above — equals that type, and the
  impl's type bounds are satisfied by that same instantiation. Matching the
  implemented type determines the type and const values; this is not a call
  and is not const argument inference at a use site. A const parameter MUST
  occur in the implemented type so matching determines it; an impl whose
  const parameter does not occur is ill-formed (`E0817`). An impl that cannot
  be so instantiated, or whose bounds are not satisfied, contributes
  **nothing** for that type: no members through member-call resolution, and
  no trait implementation for explicit dispatch or bound checking. Two impls
  that can apply to the same type after substitution conflict under the
  ordinary `E0805` rule. Satisfaction is recursive: an impl whose bounds name
  other traits applies only when those bounds hold in turn — which, because
  traits are never parameterized, always means a bound naming a trait with no
  arguments.

Replace the `#Derive` generic-type bullet with:

- `#Derive` on a type that declares generic parameters is `E0112`: the
  language has no bound that means "is a base type or implements `ToString`",
  so the expansion cannot be written for an open parameter. A generic type
  keeps its hand-written impl.
