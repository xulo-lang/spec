# Enums — Delta: Const Parameters

> **Delta** — replaces the generic-enum bullet in `types/enums.md`; the rest
> is unchanged.

Replace the generic-enum bullet with:

- An enum is generic when a generic parameter list follows its name — for
  example `enum Either<L, R> { Left(L), Right(R) }` or
  `enum Packet<const N: int> { Fixed(Array<int, N>) }` — and the type and
  const parameters are then in scope in every payload type.
