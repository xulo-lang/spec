# Composite Types — Delta: Collection Members

> **Delta** — adds the inherent-members sentence after the chapter
> introduction and replaces the List, Map, and Set sections of
> `types/composite-types.md`; the struct, optional, union, intersection,
> alias, and tuple sections are unchanged.

Composite types are built from other types. They fall into three groups: the
built-in collections `List<T>`, `Map<K, V>`, and `Set<T>`; the record types —
nominal `struct` records and positional tuples; and the constructors that
combine types — `T?`, `T | U`, and `T & U`. This chapter specifies the values
each constructor admits, how values of those types are written, and the rules
by which values move between them.

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

- A list literal `[e1, e2, …]` has type `List<T>`, where `T` is the union of the element types: `[1, "two"]` is `List<int | string>`.
- The element type of an empty literal `[]` is determined by its context — an annotation, or the uses of the binding elsewhere in the enclosing function body. If nothing determines it, a compile-time error is reported (`E0216`).
- The prefix spread `...` MAY appear only inside a list literal or the brace form of a map literal. In a list literal its operand MUST be a `List<U>` and its elements are spliced in place; in a map literal its operand MUST be a `Map<K, V>` and its entries are merged, with the later occurrence of a duplicate key winning.
- `+` concatenates two lists into a new list of type `List<T | U>` — exactly `List<T>` when both operands are `List<T>` — and leaves the operands unchanged.
- `xs[i]` requires `i` to have type `int`. Reading yields the element at that index; writing (`xs[i] = v`) stores a value of the element type into it, subject to the mutable-place rules. An index that is negative or at or past the end of the list is a runtime error for both.
- Iteration is written `for x in xs` and visits the elements in order.
- The type has these inherent members:
  - `.len` — a read-only property of type `int`, the number of elements,
    O(1);
  - `.push(v)` — append `v`; mutating; returns `unit`;
  - `.pop()` — remove and return the last element, or `null` when the list is
    empty; mutating; type `T?`;
  - `.shift()` — remove and return the first element, or `null` when the list
    is empty; mutating; type `T?`;
  - `.unshift(v)` — insert `v` at the front; mutating; returns `unit`;
  - `.splice(start, deleteCount, items: List<T> = [])` — remove
    `deleteCount` elements at `start`, insert the elements of `items` there,
    and return the removed elements as a new `List<T>` in order; mutating.
    `start` is in `0 …= len`, and `deleteCount` in `0 …= len − start`;
    outside is a runtime failure. `items` defaults to the empty list, so
    `splice(start, n)` only deletes and `splice(start, 0, items)` only
    inserts;
  - `.resize(n, fill)` — set the length to `n`: shrink from the end when
    `n < len`, append `fill` when `n > len`; mutating; returns `unit`. `n`
    MUST be non-negative; a negative `n` is a runtime failure;
  - `.truncate(n)` — remove elements from the end until the length is `n`,
    with no effect when `n >= len`; mutating; returns `unit`. `n` MUST be
    non-negative; a negative `n` is a runtime failure;
  - `.clear()` — remove all elements; mutating; returns `unit`;
  - `.slice(start, end: int? = null)` — a new `List<T>` of the elements at
    `start … end − 1`, where `null` `end` means `len`; `start` and `end` are
    in `0 …= len` with `start <= end`; outside is a runtime failure;
  - `.indexOf(v)` — the first index whose element is `== v`, or `-1`; type
    `int`;
  - `.lastIndexOf(v)` — the last such index, or `-1`; type `int`;
  - `.includes(v)` — whether some element is `== v`; type `bool`;
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
  - `.get(k)` — the value for `k`, or `null` when absent; type `V?`;
  - `.set(k, v)` — replace the value for `k`, or insert the entry; mutating;
    returns `unit`;
  - `.has(k)` — whether `k` is present; type `bool`;
  - `.remove(k)` — remove the entry and return the value it had, or `null`
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
let a = counts.get("a")                  // 1
let missing = counts.get("z")            // null: absent
counts.set("a", 3)                       // counts now holds "a" → 3
let had = counts.remove("b")             // 2
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
  the number of elements, O(1)), `.has(v)` (membership; type `bool`),
  `.insert(v)` (add `v`; returns `true` when it was not present), and
  `.remove(v)` (remove `v`; returns `true` when it was present). The mutating
  members require a mutable place.
- A subscript or member entry form does not exist: `s[v]` is `E0215` and
  `s.v` is `E0102`.
- A set has no order: the language specifies no iteration order for `Set<T>`, and programs MUST NOT depend on one.
