# Composite Types — Delta: Struct Composition Cycle

> **Delta** — adds one bullet to the “struct” bullet list in
> `types/composite-types.md`, after the “Structs are nominal …” bullet;
> every other bullet is unchanged.

## struct

- Fields are declared as `name: type`. Construction uses named arguments: `User(name: "Alice", age: 30)`.
- Every field whose type is not optional MUST be supplied at construction (`E0205` otherwise). A field of an optional type (`T?`) MAY be omitted, in which case its value is `null`.
- Structs are nominal. Two `struct` declarations with identical fields are still different types, and neither is assignable to the other. A `struct` type and a `Map` are likewise distinct: there is no implicit conversion in either direction.
- A struct's field types compose into the struct's value: the declaring type MUST NOT be reachable from its own fields through field positions, tuple elements, and alias expansion without an intervening indirection — an indirection being `List<T>`, `Map<K, V>`, `Set<T>`, `T?`, a function type, or an enum type. `struct Node { node: Node }`, the mutual pair `struct A { b: B }` / `struct B { a: A }`, and `struct S { t: (S, int) }` are compile-time errors (`E0113`). A path through one of the listed indirections breaks the cycle: `struct Node { next: Node?, children: List<Node> }` is well-formed, and recursion through enum payloads is permitted in general (see [`enums.md`](enums.md)).
- Field access is written `u.name`. Assigning to a field requires the binding to be a mutable place (a `let mut` binding or a `mut` parameter); assigning through an immutable binding is a compile-time error (`E0401`).
- Fields are private by default. A field marked `pub` is readable and writable from outside the module that declares the struct; reading, writing, or constructing a private field from another module is a compile-time error (`E0602`). A `pub struct` declaration is exported from its module (see [`../modules/README.md`](../modules/README.md)).
- Structs do not inherit: there is no base struct, no struct-to-struct subtyping, and no overriding. Shared behavior is attached with `impl` blocks and `trait` implementations (see [`../functions.md`](../functions.md) and [`traits.md`](traits.md)).
- A struct MAY be generic: `struct Pair<A, B> { first: A, second: B }` declares `Pair<int, string>` and friends (see [`generics.md`](generics.md)).
