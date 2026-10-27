# Built-ins — Delta: Members Are Not Prelude Names

> **Delta** — replaces the introduction, the two-layer table, the inner-scope
> sentence of Availability, and the prelude item of the resolution order in
> `builtins/README.md`; the rest is unchanged.

Some names belong to the language rather than to any file: the built-in types
and their members, the conversion and output functions, and the members of
the mathematical, time, and task namespaces. None of them is declared, none
of them is imported, and together they form the constant background against
which every module is read. The standard library is a different thing
entirely: library modules and their APIs are out of scope for this
specification (see [`../README.md`](../README.md), §1.1).

## Two layers

| Layer | Contains | Chapter |
|-------|----------|---------|
| **Prelude** | the built-in types, the built-in protocol `ToString`, and the built-in namespaces — in scope in every module | [`prelude.md`](prelude.md) |
| **Intrinsics** | the compiler-provided functions `print`, `println`, and `str`, together with the members of `Math`, `Time`, and `Task`, each with a defined signature | [`intrinsic-functions.md`](intrinsic-functions.md) |

The prelude decides *which names* every module has; the intrinsic chapter
gives the signature and the meaning of each function among them. A name may
belong to both layers: `print` is a name that is in scope because of the
prelude and whose signature is that of an intrinsic.

The members of `List`, `Map`, `Set`, and `string` are part of their types
([`../types/composite-types.md`](../types/composite-types.md),
[`../types/primitive-types.md`](../types/primitive-types.md)); they are not
prelude names and enter resolution only after a receiver or a type name
(`Map::fromEntries`), so they neither reserve module-scope names nor collide
with local bindings.

## Availability

- **No `use` is required.** A module that `use`s nothing still has every
  name of both layers, from its first line to its last.
- **A module-level declaration MUST NOT bear a prelude or intrinsic name.**
  Declaring `fn print(...)`, `struct Task { … }`, `type Map<K, V> = …`, or
  `let str = …` at file scope is a compile-time error (`E0106`). The complete list of
  reserved names is given in
  [prelude.md](prelude.md#reserved-names), and the diagnostic category for a
  violation is defined in [`../type-system/errors.md`](../type-system/errors.md).
- **Inner scopes behave differently.** An inner-scope `let` MAY shadow an
  intrinsic name; the name then denotes the local binding until the scope
  ends ([`../names.md`](../names.md)). The precise rules are stated in
  [Shadowing](prelude.md#shadowing). A built-in member is unaffected: it is
  reached after a receiver and no local binding hides it.

## Resolution order

When a name is used, [`../names.md`](../names.md) searches the following
scopes in order, and the first scope that provides the name wins:

1. bindings of the innermost enclosing block, innermost first;
2. parameters and type parameters of the enclosing function or closure;
3. the remaining bindings of that function, then the module's own top-level
   declarations;
4. names brought into the module by `mod`, `use`, and `pub use`;
5. the prelude — the built-in type names and the intrinsics (`print`,
   `println`, `str`, `Math`, `Time`, `Task`).

A namespace is resolved as an ordinary name *before* member access: in
`Math::max(a, b)` the leading `Math` is found by the search above, and only
then is `max` looked up as its member. Prelude names are therefore visible
only when nothing in the earlier scopes matches, which is why a local binding
of the same name hides them for as long as it is in scope.
