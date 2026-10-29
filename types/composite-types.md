# Composite Types

Composite types are built from other types. They fall into three groups: the
built-in collections `List<T>`, `Array<T, N>`, `Map<K, V>`, and `Set<T>`; the
record types — nominal `struct` records and positional tuples; and the
constructors that combine types — `T?`, `T | U`, and `T & U`. This chapter
specifies the values each constructor admits, how values of those types are
written, and the rules by which values move between them.

The built-in collections expose **inherent members** — read-only properties
and methods — listed in their sections. They are part of the type, not prelude
operations, and a program cannot add an inherent member; a trait
implementation for a built-in type is still an ordinary trait implementation
([`../functions.md`](../functions.md)).

## List<T>

A `List<T>` is an ordered, growable sequence — a dynamic array — of elements, all of the same type `T`, addressed by a 0-based index.

```xulo
let xs = [1, 2, 3]          // List<int>
let mixed = [1, "two"]      // List<int | string>
let names: List<string> = []
```

- Absent an expected `Array` type, a list literal `[e1, e2, …]` has type
  `List<T>`, where `T` is the union of the element types: `[1, "two"]` is
  `List<int | string>`. Checked against an expected `Array<T, N>`, the same
  syntax is an array literal instead (see `## Array<T, N>` below).
- The element type of an empty literal `[]` is determined by its context — an annotation, or the uses of the binding elsewhere in the enclosing function body. If nothing determines it, a compile-time error is reported (`E0216`).
- The prefix spread `...` MAY appear only inside a list literal or the brace form of a map literal. In a list literal its operand MUST be a `List<U>` and its elements are spliced in place; in a map literal its operand MUST be a `Map<K, V>` and its entries are merged, with the later occurrence of a duplicate key winning.
- `+` concatenates two lists into a new list of type `List<T | U>` — exactly `List<T>` when both operands are `List<T>` — and leaves the operands unchanged.
- `xs[i]` requires `i` to have type `int`. Reading yields the element at that index; writing (`xs[i] = v`) stores a value of the element type into it, subject to the mutable-place rules. An index that is negative or at or past the end of the list is a runtime error for both.
- Iteration is written `for x in xs` and visits the elements in order.
- The type has these inherent members:
  - `.len` — a read-only property of type `int`, the number of elements,
    O(1);
  - `.push(v: T)` — append `v`; mutating; returns `unit`;
  - `.pop()` — remove and return the last element, or `null` when the list is
    empty; mutating; type `T?`;
  - `.shift()` — remove and return the first element, or `null` when the list
    is empty; mutating; type `T?`;
  - `.unshift(v: T)` — insert `v` at the front; mutating; returns `unit`;
  - `.splice(start: int, deleteCount: int, items: List<T> = [])` — remove
    `deleteCount` elements at `start`, insert the elements of `items` there,
    and return the removed elements as a new `List<T>` in order; mutating.
    `start` is in `0 …= len`, and `deleteCount` in `0 …= len − start`;
    outside is a runtime failure. `items` defaults to the empty list, so
    `splice(start, n)` only deletes and `splice(start, 0, items)` only
    inserts;
  - `.resize(n: int, fill: T)` — set the length to `n`: shrink from the end when
    `n < len`, append `fill` when `n > len`; mutating; returns `unit`. `n`
    MUST be non-negative; a negative `n` is a runtime failure;
  - `.truncate(n: int)` — remove elements from the end until the length is `n`,
    with no effect when `n >= len`; mutating; returns `unit`. `n` MUST be
    non-negative; a negative `n` is a runtime failure;
  - `.clear()` — remove all elements; mutating; returns `unit`;
  - `.slice(start: int, end: int? = null)` — a new `List<T>` of the elements at
    `start … end − 1`, where `null` `end` means `len`; `start` and `end` are
    in `0 …= len` with `start <= end`; outside is a runtime failure;
  - `.indexOf(v: T)` — the first index whose element is `== v`, or `-1`; type
    `int`;
  - `.lastIndexOf(v: T)` — the last such index, or `-1`; type `int`;
  - `.includes(v: T)` — whether some element is `== v`; type `bool`;
  - `.join(separator: string = ",")` — the elements' string forms joined by
    `separator`, with `""` for an empty list; type `string`. The element type
    MUST be a base type or implement `ToString` (the `str` domain); otherwise
    the call is a compile-time error (`E0212`);
  - `.reverse()` — reverse the list in place; mutating; returns `unit`;
  - `.entries()` — a `List<(int, T)>` of `(index, element)` pairs, in order;
  - `.forEach(f: fn(T): unit)` — call `f` on each element in order; returns
    `unit`;
  - `.map<U>(f: fn(T): U)` — a new `List<U>` of the callback results, in
    order;
  - `.filter(f: fn(T): bool)` — a new `List<T>` of the elements for which `f`
    returns `true`, in order;
  - `.find(f: fn(T): bool)` — the first element for which `f` is `true`, or
    `null`; type `T?`;
  - `.findIndex(f: fn(T): bool)` — the index of that element, or `-1`; type
    `int`;
  - `.some(f: fn(T): bool)` — whether `f` is `true` for any element; `false`
    for an empty list;
  - `.every(f: fn(T): bool)` — whether `f` is `true` for every element; `true`
    for an empty list;
  - `.reduce<U>(f: fn(U, T): U, initial: U)` — fold the elements in order,
    starting from `initial`; an empty list returns `initial`; type `U`;
  - `.sort(f: fn(T, T): int)` — a stable in-place sort. `f(a, b)` returns a
    negative value when `a` sorts before `b`, zero when they are equal, and a
    positive value otherwise; mutating; returns `unit`.
  The mutating members require the receiver to be a mutable place.
  `indexOf`, `lastIndexOf`, and `includes` require `T` to admit `==`; `NaN`
  never equals itself, so it is never found.
- The callback members take a synchronous Xulo function value — `fn(T): U`,
  `fn(T): bool`, `fn(U, T): U`, or `fn(T, T): int` as the member states. The
  callback receives the element only, not its index (use `.entries()` for
  index/value pairs). `forEach`/`map`/`filter`/`reduce` visit every element;
  `find`/`findIndex`/`some`/`every` stop at the first decisive element. The
  receiver is borrowed for the duration of the call, so a callback cannot
  mutate it; an ordinary borrow conflict is reported (`E0404`/`E0405`). A
  callback that panics stops the call; the callback members do not return
  `Result`. `sort` order is unspecified for a comparator that does not define
  a consistent ordering.
- The type system does not assign identity to list values: binding, assignment, argument passing, and return never depend on whether two `List<T>` values share storage. Storage sharing, `shared` state, and aliasing are governed by the memory model, not by the type of the list (see [`../memory-and-runtime.md`](../memory-and-runtime.md)).

```xulo
let head = [1, 2]
let tail = [3, 4]
let all = [...head, ...tail]   // List<int>
let more = all + [5]           // List<int>

let mut sum = 0
for x in all {
  sum = sum + x
}
```

## Array<T, N>

`Array<T, N>` is a built-in **inline, fixed-length value type**: `T` is a type
parameter, `N` is a const parameter of type `int`, and N MUST be non-negative;
a concrete negative N is `E0814`. Every value holds exactly N elements of `T`,
laid out inline; the type is invariant in `T` and exact in N.

- Binding, assignment, argument passing, and return copy the Array value;
  each of its N elements is copied by its own category (a `List` element is
  shared, a `struct` element is value-copied). An `Array` has no backing-store
  identity and no reference sharing.
- `a[i]` and `a[i] = v` require an `int` index in `0 … N − 1`; an index outside
  that range is a runtime failure. `for x in a` visits the elements in index
  order.
- `.len` is a read-only `int` property equal to N, O(1).
- The read, callback, and sort members are the List members — `.slice`,
  `.indexOf`, `.lastIndexOf`, `.includes`, `.join`, `.entries`, `.reverse`,
  `.sort`, `.forEach`, `.map`, `.filter`, `.find`, `.findIndex`, `.some`,
  `.every`, `.reduce` — with these differences:
  - `.slice` and `.filter` return `List<T>`; `.map` returns `Array<U, N>`
    because the length is preserved;
  - `.reverse` and `.sort` mutate the array; as in the List rule, the
    mutating members require the receiver to be a mutable place (`E0404` on
    an immutable binding); `sort` is stable and takes a comparator;
  - callbacks are synchronous, receive the element only, and cannot mutate
    the receiver (the ordinary borrow rules, `E0404`/`E0405`).
- There is **no** `.push`, `.pop`, `.shift`, `.unshift`, `.splice`, `.resize`,
  `.truncate`, or `.clear`: the length is N. A member outside this set is
  `E0102`.
- A list literal checked against an expected `Array<T, N>` is an array
  literal: its element count MUST equal N, and its elements check against T in
  order. `[]` is the N = 0 case; a length mismatch is `E0815`, and a literal
  that cannot be checked statically — a spread, or an abstract const parameter
  N — is `E0818`. Without an expected Array type, `[1, 2, 3]` remains a
  `List<int>`.
- `Array::filled<T, const N: int>(value: T): Array<T, N>` is the generic
  constructor: it creates N copies of `value` and is written with explicit type
  and const arguments, as in `Array::filled<int, 3>(0)` — there is no const
  inference. A call that omits the list is `E0816` and a partial list is
  `E0808` (the const-generic rules). It is the only constructor besides the
  context-typed literal.
- `Array<T, N>` and `List<T>` are unrelated nominal types: there is no
  implicit conversion in either direction, and no FFI boundary for Array in
  version 1.
- An `Array` has no string form: `str(a)` and `${a}` are `E0212`, as for any
  other collection operand of the intrinsics.

## Map<K, V>

A `Map<K, V>` is an insertion-ordered dictionary: a sequence of key–value pairs, ordered by insertion, in which each key appears at most once.

A map literal has two forms:

- **The brace form** `{ k: v, … }` — keys are identifiers, each becoming the
  string key of the entry. Its default type is `Map<string, C>`, where `C` is
  the union of the value types: `{ name: "lyy", age: 30 }` is
  `Map<string, string | int>`. With an expected type `Map<K, V>` in scope,
  every value checks against `V` and `K` MUST admit `string`. A duplicate key
  in one literal is a compile-time error (`E0223`). The empty literal `{}` determines
  its type from context and is a compile-time error (`E0216`) when nothing
  determines it, exactly like `[]`.
- **The typed form** `Map<K, V>{ key: value, … }` — keys are expressions of
  the hashable type `K` — constructs a value whose static type is exactly
  `Map<K, V>`; with no entries it constructs an empty map. It is written only
  where no expected map type is in effect, or where a key is not an
  identifier: when an expected `Map<K, V>` is in scope and every key is an
  identifier (or the literal is empty), the brace form MUST be used, and the
  typed form repeats the context's type — a compile-time error (`E0221`,
  see [`../expressions/literals.md`](../expressions/literals.md)).

```xulo
let counts = { "a": 1, "b": 2 }                 // Map<string, int>, inferred
let user = { name: "lyy", age: 30 }             // Map<string, string | int>
let same: Map<string, int> = { a: 1, b: 2 }     // brace form, expected type applied
let empty: Map<string, int> = {}                // entry type from the annotation
let ages = Map<int, string>{ 30: "thirty" }     // typed form: keys are not identifiers
```

- The key type `K` MUST be hashable. The hashable types are `string`, `int`,
  `float`, `bool`, and enum types whose variants carry no payload. Any
  other key type is a compile-time error (`E0224`).
- **Key identity.** Two keys that are `==` name the same entry. `float` is
  the exception: IEEE-754 `NaN` never equals itself, yet every `NaN` key
  names one and the same entry — a write under one `NaN` key overwrites the
  entry stored under any other — and `-0.0` and `0.0` name the same entry. A
  hash of a `float` key MUST agree with this identity.
- The value type `V` may be any type, `unknown` included.
- The type has inherent members:
  - `.size` — a read-only property of type `int`, the number of entries,
    O(1);
  - `.get(k: K)` — the value for `k`, or `null` when absent; type `V?`;
  - `.set(k: K, v: V)` — replace the value for `k`, or insert the entry;
    mutating; returns `unit`;
  - `.has(k: K)` — whether `k` is present; type `bool`;
  - `.remove(k: K)` — remove the entry and return the value it had, or `null`
    when absent; mutating; type `V?`;
  - `.keys()`, `.values()`, `.entries()` — new `List`s of the keys, the
    values, and the `(key, value)` pairs, in insertion order;
  - `Map::fromEntries(entries)` — a static constructor from a
    `List<(K, V)>`, in list order; a repeated key keeps the later entry.
  The mutating members require the receiver to be a mutable place.
- **There is no key access by subscript or by member.** `m[k]`,
  `m[k] = v`, and `m.key` do not exist. A key is read with `m.get(k)` —
  absence is `null`, not a failure — written with `m.set(k, v)`, tested with
  `m.has(k)`, and removed with `m.remove(k)`. A subscript on a map is `E0215`.
- Iteration over keys with `for … in` remains core syntax and yields the keys
  in insertion order
  ([`../expressions/control-flow.md`](../expressions/control-flow.md)); it is
  not a key access.

```xulo
let mut counts = { "a": 1, "b": 2 }      // Map<string, int>
let a = counts.get("a")                  // 1 (int?)
let missing = counts.get("z")            // null: absent
counts.set("a", 3)                       // counts now holds "a" → 3
let had = counts.remove("b")             // 2 (int?)
print(counts.size)                       // 1
print(counts.has("a"))                   // true
for k in counts {                        // keys, in insertion order
  print(k)
}
```

## Set<T>

A `Set<T>` is an unordered collection of unique values: it contains each distinct value at most once.

- The element type `T` MUST be hashable — the same set of types admitted as map keys: `string`, `int`, `float`, `bool`, and payload-free enums. Any other element type is a compile-time error (`E0225`). Element identity follows the key identity rule above: every `NaN` is one element of a `Set<float>`.
- The language defines no set literal: the brace syntax belongs to `Map` — `{}` is its empty literal — so a set is constructed with the static member `Set::fromList(values)`.
- The type has inherent members: `.size` (a read-only property of type `int`,
  the number of elements, O(1)), `.has(v: T)` (membership; type `bool`),
  `.insert(v: T)` (add `v`; returns `true` when it was not present), and
  `.remove(v: T)` (remove `v`; returns `true` when it was present). The
  mutating members require a mutable place.
- A subscript or member entry form does not exist: `s[v]` is `E0215` and
  `s.v` is `E0102`.
- A set has no order: the language specifies no iteration order for `Set<T>`, and programs MUST NOT depend on one.

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
- Every field whose type is not optional MUST be supplied at construction (`E0205` otherwise). A field of an optional type (`T?`) MAY be omitted, in which case its value is `null`.
- Structs are nominal. Two `struct` declarations with identical fields are still different types, and neither is assignable to the other. A `struct` type and a `Map` are likewise distinct: there is no implicit conversion in either direction.
- A struct's field types compose into the struct's value: the declaring type MUST NOT be reachable from its own fields through field positions, tuple elements, array element positions, and alias expansion without an intervening indirection — an indirection being `List<T>`, `Map<K, V>`, `Set<T>`, `T?`, a function type, or an enum type. `struct Node { node: Node }`, the mutual pair `struct A { b: B }` / `struct B { a: A }`, and `struct S { t: (S, int) }` are compile-time errors (`E0113`). An `Array<T, N>` is inline, so an array element position is part of that composition, not an indirection: `struct S { a: Array<S, 3> }` is `E0113`. A path through one of the listed indirections breaks the cycle: `struct Node { next: Node?, children: List<Node> }` is well-formed, and recursion through enum payloads is permitted in general (see [`enums.md`](enums.md)).
- Field access is written `u.name`. Assigning to a field requires the binding to be a mutable place (a `let mut` binding or a `mut` parameter); assigning through an immutable binding is a compile-time error (`E0401`).
- Fields are private by default. A field marked `pub` is readable and writable from outside the module that declares the struct; reading, writing, or constructing a private field from another module is a compile-time error (`E0602`). A `pub struct` declaration is exported from its module (see [`../modules/README.md`](../modules/README.md)).
- Structs do not inherit: there is no base struct, no struct-to-struct subtyping, and no overriding. Shared behavior is attached with `impl` blocks and `trait` implementations (see [`../functions.md`](../functions.md) and [`traits.md`](traits.md)).
- A struct MAY be generic: `struct Pair<A, B> { first: A, second: B }` declares `Pair<int, string>` and friends (see [`generics.md`](generics.md)).

Use a `struct` when a type needs named fields with their own types, methods, trait implementations, visibility control, or a distinct identity; use a `Map` for dynamic key–value data.

## Optional types

The optional type `T?` is shorthand for `T | null`: a value of type `T?` is either a `T` or the `null` literal.

```xulo
fn greet(name: string?): string {
  let who = name ?? "stranger"
  `Hello, ${who}!`
}

let user = { email: "a@b.c" }                 // Map<string, string>
let missing: Map<string, string>? = null
print(user?.email ?? "unknown")
print(missing?.email ?? "unknown")
```

- **Reading an optional.** A value of type `T?` MUST be checked or propagated before it is used where `T` is required. Using an optional where a non-optional type is expected — as a `string` operand of `+`, as a field of type `T`, as an argument of type `T` — is a compile-time error (`E0201`).
- A comparison against `null` narrows a `T?` binding: in a branch guarded by `x != null`, the binding `x` has type `T`, and in the opposite branch it has type `null`. The rule does not apply to `unknown` — only a `match` type pattern narrows a value of that type (see [`primitive-types.md`](primitive-types.md)). The nullish operator `a ?? b` (operands of type `T?` and `U`) has type `T | U`.

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

fn setStatus(s: Status) {
  match s {
    "active" => print("on")
    "inactive" => print("off")
    _ => print("pending")
  }
}

let which: string | int = 1
```

- Membership is tested with `match`: literal patterns test literal members, `Enum::Variant` patterns test enum members, `Struct(...)` patterns test struct members, type patterns test base-type and nominal members (`string s => …`), and `_` matches anything; `is` tests one base-type or nominal membership at a time (`x is string`) (see [`../expressions/control-flow.md`](../expressions/control-flow.md)).
- Literal types may be union members: a string, number, or boolean literal denotes the type containing exactly that value, so `"active" | "inactive"` is the type of a status field and a call site accepts either literal.
- A union containing `null` is exactly the corresponding optional type: `T | null` and `T?` are the same type (see [`type-relations.md`](type-relations.md)). Otherwise no rewriting is performed: redundant members such as `int | int` are permitted and denote the same values as `int`.
- Two nominal `struct` types MAY appear together in a union, even when their fields are identical: they remain distinct types, and `match` distinguishes them by their declared names.
- Unions MAY be used in parameter types, binding annotations, field types, and type aliases. A value of union type MUST be narrowed — by `match`, or by `is` in an `if` condition — before an operation specific to one member is applied.

## Intersection types

An intersection type `T & U` describes a value that satisfies both `T` and `U`. Intersections combine trait contracts: every operand of `&` MUST be a trait type. A value carries exactly one nominal type, so two distinct `struct` or `enum` types would intersect in nothing; traits are the contracts a single value can satisfy together, and that is what `&` combines.

```xulo
trait Area { fn area(self): int }
trait Scalable { fn scale(self, k: int): int }

fn measure(x: Area & Scalable): int { x.scale(2).area() }
```

- An intersection of trait types — `Area & Scalable` — denotes a value that implements both traits. It is an intersection *type*, written wherever a type is expected; a generic *bound* is a different thing and is written with `+` (`<T: Area + Scalable>`), never with `&` (see [`generics.md`](generics.md) and [`traits.md`](traits.md)).
- Intersections are insensitive to order and grouping: `T & U` and `U & T` are the same type, and `T & (U & V)` is the same type as `(T & U) & V`.
- Any other combination — two nominal types, a nominal type and a trait, a `Map` or `List` and anything — is a compile-time error (`E0226`).
- `&` binds tighter than `|`: `A & B | C` denotes `(A & B) | C`.

## Type aliases

`type Name = T` introduces an alias for an existing type.

```xulo
type User = Map<string, string | int>
type Status = "active" | "inactive"
type Handler = fn(string): int
type Wrapper<T> = List<T>
```

- An alias is transparent: it denotes exactly the type on its right-hand side and introduces no new type. `User` and `Map<string, string | int>` are the same type everywhere, and no `impl` block, method, or trait implementation may be declared *for an alias* — implementations attach to the underlying type (see [`traits.md`](traits.md)).
- An alias MAY be generic: `Wrapper<T>` is used by application, `Wrapper<User>`, with the type argument inferred or written where types are written (see [`generics.md`](generics.md)).
- An alias declaration that expands, through a chain of aliases, back to itself is a compile-time error (`E0105`; `type A = B` together with `type B = A`). Recursion that passes through a type constructor — `List<T>`, `Map<K, V>`, `Set<T>`, or `T?` — is not an alias cycle and is permitted: `type Json = null | bool | int | string | List<Json> | Map<string, Json>` is well-formed.
- An alias is a module-level declaration and MAY be exported with `pub` and brought into type positions with `use type` (see [`../modules/README.md`](../modules/README.md)).

## Tuples

A **tuple** is an ordered, fixed-length sequence of values with one type per
position: the positional counterpart of a `struct`, whose fields carry names
instead. Tuples model results that are inherently positional — a pair, a
coordinate, the two halves of a split.

```xulo
let p: (int, string) = (10, "ten")
let q = (1, true)                        // (int, bool)
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
fixed sequence with named fields is a `struct`; a dynamic key–value
collection is a `Map`. Reach for a tuple when the positions themselves
carry the meaning.
