# Composite Types

Composite types are built from other types. They fall into three groups: the built-in collections `list<T>`, `map<K, V>`, and `set<T>`; the record types — structural object types, nominal `struct` records, and positional tuples; and the constructors that combine types — `T?`, `T | U`, and `T & U`. This chapter specifies the values each constructor admits, how values of those types are written, and the rules by which values move between them.

## list<T>

A `list<T>` is an ordered, growable sequence of elements, all of the same type `T`, addressed by a 0-based index.

```xulo
let xs = [1, 2, 3]          // list<int>
let mixed = [1, "two"]      // list<int | string>
let names: list<string> = []
```

- A list literal `[e1, e2, …]` has type `list<T>`, where `T` is the union of the element types: `[1, "two"]` is `list<int | string>`.
- The element type of an empty literal `[]` is determined by its context — an annotation, or the uses of the binding elsewhere in the enclosing function body. If nothing determines it, a compile-time error is reported.
- The prefix spread `...` MAY appear only inside a list literal or an object literal. In a list literal its operand MUST be a `list<U>` and its elements are spliced in place; in an object literal its operand MUST be an object and its fields are merged, with the later occurrence of a duplicate key winning.
- `+` concatenates two lists into a new list of type `list<T | U>` — exactly `list<T>` when both operands are `list<T>` — and leaves the operands unchanged.
- `xs[i]` requires `i` to have type `int`. Reading yields the element at that index; writing (`xs[i] = v`) stores a value of the element type into it, subject to the mutable-place rules. An index that is negative or at or past the end of the list is a runtime error for both.
- Iteration is written `for x in xs` and visits the elements in order.
- The type system does not assign identity to list values: binding, assignment, argument passing, and return never depend on whether two `list<T>` values share storage. Storage sharing, `shared` state, and aliasing are governed by the memory model, not by the type of the list (see [`../memory-and-runtime.md`](../memory-and-runtime.md)).

```xulo
let head = [1, 2]
let tail = [3, 4]
let all = [...head, ...tail]   // list<int>
let more = all + [5]           // list<int>

let mut sum = 0
for x in all {
  sum = sum + x
}
```

## map<K, V>

A `map<K, V>` is an insertion-ordered dictionary: a sequence of key–value pairs, ordered by insertion, in which each key appears at most once.

```xulo
let empty: map<string, int> = map<string, int>{ }
let counts: map<string, int> = map<string, int>{ "a": 1, "b": 2 }
```

- A map literal has the form `map<K, V>{ key: value, … }` and constructs a value whose static type is `map<K, V>`; with no fields it constructs an empty map.
- The key type `K` MUST be hashable. The hashable types are `string`, `int`, `float`, `boolean`, and enum types whose variants carry no payload. Any other key type is a compile-time error.
- The value type `V` may be any type.
- A map is not an object, and an object is not a map. The object literal `{ name: "x" }` has a structural object type with one field; it is never a `map<string, string>`, and there is no implicit conversion in either direction. Moving data between the two uses the conversion operations of the prelude (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- Reading and writing entries with a subscript (`counts["a"]`, `counts["a"] = 1`)
  and iteration over keys with `for … in` are core syntax (see
  [`../expressions/path-and-access.md`](../expressions/path-and-access.md) and
  [`../expressions/control-flow.md`](../expressions/control-flow.md)). All
  other map operations — removal, size, entry tests, bulk conversion — are
  prelude functions (see [`../builtins/prelude.md`](../builtins/prelude.md)).

```xulo
let record = { name: "x" }                      // object type: fields are static
let table: map<string, string> = map<string, string>{ "name": "x" }
```

## set<T>

A `set<T>` is an unordered collection of unique values: it contains each distinct value at most once.

- The element type `T` MUST be hashable — the same set of types admitted as map keys: `string`, `int`, `float`, `boolean`, and payload-free enums. Any other element type is a compile-time error.
- The language defines no set literal. Sets are constructed with the prelude constructor and manipulated with the prelude set operations (see [`../builtins/prelude.md`](../builtins/prelude.md)).
- A set has no order: the language specifies no iteration order for `set<T>`, and programs MUST NOT depend on one.

## object

An object type is an anonymous structural record: a set of named fields, each with its own type. Object types are structural — they are defined by their fields, not by a declared name.

```xulo
let user = { name: "lyy", age: 30 }   // { name: string, age: int }
let name = user.name                  // string
let age = user.age                    // int

type Person = { name: string, age: int }
let p: Person = user
```

- An object literal `{ f1: e1, f2: e2 }` has the structural type listing each field with the type of its initializer. A duplicate field name in one literal is a compile-time error.
- Field access `e.f` has the type of the field. Accessing a field that the static type of `e` does not declare is a compile-time error.
- Two object types are the same type when they declare the same field names with the same types; field order does not matter. `Person` above and the type of `user` are the same type — the alias adds no identity.
- Object types are width-subtyped: a value with *more* fields MAY be used where a type with fewer fields is required, provided every required field is present with an assignable type.

```xulo
fn area(p: { x: int, y: int }): int { p.x * p.y }

let origin = { x: 10, y: 20, label: "origin" }
area(origin)          // OK: extra fields are allowed
```

- `object` is the open record type: it stands for "some object, its fields unknown", and every object value is assignable to it. Member access on a value whose static type is `object` is a compile-time error — the field set is not statically known — so annotate a typed object type (or use a `struct`) whenever fields must be read.
- Assigning to a field (`p.x = 5`) requires the object binding to be a mutable place — a `let mut` binding or a `mut` parameter — and is otherwise a compile-time error (see [`../memory-and-runtime.md`](../memory-and-runtime.md)).
- Object types have no field visibility markers: every field of an object type is accessible wherever the type itself is accessible.

## struct

A `struct` declares a nominal record type: a named type with explicit fields.

```xulo
struct User {
  name: string
  age: int
  email: string?
}

let alice = User(name: "Alice", age: 30)
let mut bob = User(name: "Bob", age: 25, email: "bob@example.com")
bob.age = 26
```

- Fields are declared as `name: type`. Construction uses named arguments: `User(name: "Alice", age: 30)`.
- Every field whose type is not optional MUST be supplied at construction. A field of an optional type (`T?`) MAY be omitted, in which case its value is `null`.
- Structs are nominal. Two `struct` declarations with identical fields are still different types, and neither is assignable to the other. A `struct` type and a structural object type are likewise distinct: there is no implicit conversion in either direction.
- Field access is written `u.name`. Assigning to a field requires the binding to be a mutable place (a `let mut` binding or a `mut` parameter); assigning through an immutable binding is a compile-time error.
- Fields are private by default. A field marked `pub` is readable and writable from outside the module that declares the struct; reading, writing, or constructing a private field from another module is a compile-time error. A `pub struct` declaration is exported from its module (see [`../modules/README.md`](../modules/README.md)).
- Structs do not inherit: there is no base struct, no struct-to-struct subtyping, and no overriding. Shared behavior is attached with `impl` blocks and `trait` implementations (see [`../functions.md`](../functions.md) and [`traits.md`](traits.md)).
- A struct MAY be generic: `struct Pair<A, B> { first: A, second: B }` declares `Pair<int, string>` and friends (see [`generics.md`](generics.md)).

### Structs compared with object types

| Feature | `struct` | object type (`type` alias of `{ … }`) |
|---------|----------|----------------------------------------|
| Declared name | yes — the declaration introduces a nominal type | none — the type is its fields |
| Identity | nominal: identical fields still give distinct types | structural: identical fields give the same type |
| Construction | `User(name: "Alice", age: 30)`, named arguments | object literal `{ name: "Alice", age: 30 }` |
| Field visibility | private by default; `pub` per field | every field is accessible where the type is |
| `impl` methods and traits | attached to the struct | attached to the underlying type, never to the alias |
| Conversion to the other | none, in either direction | none, in either direction |

Use a `struct` when a type needs methods, trait implementations, visibility control, or a distinct identity; use an object type for a simple structural data shape.

## Optional types

The optional type `T?` is shorthand for `T | null`: a value of type `T?` is either a `T` or the `null` literal.

```xulo
fn greet(name: string?): string {
  let who = name ?? "stranger"
  `Hello, ${who}!`
}

let user = { email: "a@b.c" }
let missing: { email: string }? = null
print(user?.email ?? "unknown")
print(missing?.email ?? "unknown")
```

- **Reading an optional.** A value of type `T?` MUST be checked or propagated before it is used where `T` is required. Using an optional where a non-optional type is expected — as a `string` operand of `+`, as a field of type `T`, as an argument of type `T` — is a compile-time error.
- A comparison against `null` narrows the type: in a branch guarded by `x != null`, the binding `x` has type `T`; in the opposite branch it has type `null`. The nullish operator `a ?? b` (operands of type `T?` and `U`) has type `T | U`.

```xulo
fn describe(v: string?): string {
  if v != null {
    "value: " + v      // v is narrowed to string here
  } else {
    "absent"
  }
}
```
- **Optional chaining.** `x?.f` evaluates `x`; if `x` is `null`, the expression is `null` and the member is not read, otherwise the expression is the value of the member. `?.` composes with the rest of the postfix chain — including calls — so a chain such as `a?.b?.c` stops at the first `null` and nothing to the right of that `null` is evaluated.
- If `x` has type `T?` and the member `f` has type `U`, then `x?.f` has type `U?`: every `?.` step adds one optional layer, and an optional chain is consumed with `??`, a `null` test, or a ternary. When `U` is itself optional the result is a nested optional, which the non-collapse rule below keeps as two layers.
- **Nested optionals do not collapse.** `T??` is a distinct type from `T?`; it is not rewritten to `T?`, and each layer MUST be stripped by its own check: narrowing a `T??` once yields `T?`, and narrowing it again yields `T`.
- `null` is assignable to `T?` for every `T`, and to no other type (see [`primitive-types.md`](primitive-types.md)).
- **Absence is not an error.** A `T?` carries no reason and no error value;
  operations that can fail for a reason return `Result<T, E>`, and the two
  compose as `Result<T?, E>` when both are possible
  ([`../error-handling.md`](../error-handling.md#absence-and-failure-are-orthogonal)).

## Union types

A union type `T | U` describes a value that is a `T` or a `U`.

```xulo
type Status = "active" | "inactive" | "pending"

fn set_status(s: Status) {
  match s {
    "active" => print("on")
    "inactive" => print("off")
    _ => print("pending")
  }
}

let which: string | int = 1
```

- Membership is tested with `match`: literal patterns test literal members, `Enum::Variant` patterns test enum members, `Struct(...)` patterns test struct members, and `_` matches anything (see [`../expressions/README.md`](../expressions/README.md)).
- Literal types may be union members: a string, number, or boolean literal denotes the type containing exactly that value, so `"active" | "inactive"` is the type of a status field and a call site accepts either literal.
- A union containing `null` is exactly the corresponding optional type: `T | null` and `T?` are the same type (see [`type-relations.md`](type-relations.md)). Otherwise no rewriting is performed: redundant members such as `int | int` are permitted and denote the same values as `int`.
- Two nominal `struct` types MAY appear together in a union, even when their fields are identical: they remain distinct types, and `match` distinguishes them by their declared names. Two structural object types with identical fields *are* the same type, so listing both lists the same member twice.
- Unions MAY be used in parameter types, binding annotations, field types, and type aliases. A value of union type MUST be narrowed — by `match` or an explicit test — before an operation specific to one member is applied.

## Intersection types

An intersection type `T & U` describes a value that satisfies both `T` and `U`. Intersections combine structural object shapes and trait contracts.

```xulo
type Entity = { id: int }
type Named = { name: string }
type Person = Entity & Named     // { id: int, name: string }
```

- The intersection of object types has the fields of every member. If the same field appears in more than one member, it MUST be declared with the same type in each; otherwise a compile-time error is reported.
- Intersections are insensitive to order and grouping: `T & U` and `U & T` are the same type, and `T & (U & V)` is the same type as `(T & U) & V`.
- An intersection of trait types — `Area & Scalable` — denotes a value that implements both traits. It is an intersection *type*, written wherever a type is expected; a generic *bound* is a different thing and is written with `+` (`<T: Area + Scalable>`), never with `&` (see [`generics.md`](generics.md) and [`traits.md`](traits.md)).
- Two distinct nominal types — two different `struct` or `enum` declarations — MUST NOT be combined by `&`; no value can be both, and the intersection is a compile-time error.
- `&` binds tighter than `|`: `A & B | C` denotes `(A & B) | C`.

## Type aliases

`type Name = T` introduces an alias for an existing type.

```xulo
type User = { name: string, age: int }
type Status = "active" | "inactive"
type Handler = fn(string): int
type ApiResponse<T> = { data: T?, error: string? }
```

- An alias is transparent: it denotes exactly the type on its right-hand side and introduces no new type. `User` and `{ name: string, age: int }` are the same type everywhere, and no `impl` block, method, or trait implementation may be declared *for an alias* — implementations attach to the underlying type (see [`traits.md`](traits.md)).
- An alias MAY be generic: `ApiResponse<T>` is used by application, `ApiResponse<User>`, with the type argument inferred or written where types are written (see [`generics.md`](generics.md)).
- An alias declaration that expands, through a chain of aliases, back to itself is a compile-time error (`type A = B` together with `type B = A`). Recursion that passes through a type constructor — `list<T>`, `map<K, V>`, `set<T>`, `T?`, or an object or struct field — is not an alias cycle and is permitted: `type Tree = { children: list<Tree> }` is well-formed.
- An alias is a module-level declaration and MAY be exported with `pub` and imported with `import type` (see [`../modules/README.md`](../modules/README.md)).

## Tuples

A **tuple** is an ordered, fixed-length sequence of values with one type per
position: the positional counterpart of a `struct`, whose fields carry names
instead. Tuples model results that are inherently positional — a pair, a
coordinate, the two halves of a split.

```xulo
let p: (int, string) = (10, "ten")
let q = (1, true)                        // (int, boolean)
let first = p.0                          // 10
fn split(s: string): (string, string) { (s, s) }
```

- **Type.** A tuple type is written `(T₁, T₂, …)` and has **at least two**
  elements. Parentheses without a comma only group: `(T)` is not a
  one-element tuple, and there is no empty tuple type — `unit` is never
  written `()`.
- **Literal.** A tuple literal is `(e₁, e₂, …)`, at least two elements, a
  trailing comma allowed. Its type is the tuple of the element types; with an
  expected tuple type in scope each element checks against the expected
  element type
  ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
- **Positional access.** `p.i` reads element `i`, counting from `0`, and
  `p?.i` is the `null`-tolerant form; the index MUST name an existing
  position, otherwise `E0220`. The rules are in
  [`../expressions/path-and-access.md`](../expressions/path-and-access.md).
- **Destructuring.** `let (a, b) = p` binds the elements positionally; the
  name count MUST equal the tuple's arity, otherwise `E0219`. Each name
  binds immutably (`let mut (a, b)` does not exist), and `_` may stand in
  for an ignored element
  ([`../statements/let-and-assignment.md`](../statements/let-and-assignment.md)).
- **Writing elements.** Through a mutable binding: `let mut r = (1, 2)` then
  `r.0 = 5`, under the ordinary mutable-place rules
  ([`../memory-and-runtime.md`](../memory-and-runtime.md)). Because elements
  may be written, the tuple type is **invariant** in its element types
  ([`type-relations.md`](type-relations.md)).
- **Value semantics.** A tuple is a value: binding or assigning copies it, and
  `==` compares element-wise for tuples of the same arity. Different arities
  have no common type, so `(1, 2) == (1, 2, 3)` is a compile-time error
  (`E0211`) rather than `false`.
- **What tuples do not have.** No iteration (`for x in p` is `E0213`) — the
  arity is not a member, so there is no `.len` either — no prefix spread in
  a literal, and no tuple patterns for `match` in this version: read the
  elements and `match` on those.

Tuples are structural: `(int, string)` is the same type wherever it is
written, and a `type` alias may name it (`type Pair = (int, string)`). A
fixed sequence with named fields is a `struct`; an object is a record with
keys and width subtyping. Reach for a tuple when the positions themselves
carry the meaning.
