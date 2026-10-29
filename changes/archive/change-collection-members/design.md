# Design: built-in collection and string members

## The decision

The built-in collections and `string` expose a closed set of inherent
members — read-only properties, instance methods, and static constructors:

| Receiver | Members | Result |
|----------|---------|--------|
| `List<T>` | `.len` (property) | `int`, O(1) |
| | `.push(v)` / `.pop()` / `.shift()` / `.unshift(v)` | `unit` / `T?` / `T?` / `unit`; mutating |
| | `.splice(start, deleteCount, items = [])` | `List<T>` removed; mutating |
| | `.resize(n, fill)` / `.truncate(n)` / `.clear()` | `unit`; mutating |
| | `.slice(start, end?)` | `List<T>`; new list |
| | `.indexOf(v)` / `.lastIndexOf(v)` / `.includes(v)` | `int` / `int` / `bool`; `==` |
| | `.join(separator = ",")` | `string`; element string form |
| | `.reverse()` | `unit`; mutating |
| | `.entries()` | `List<(int, T)>`, in order |
| | `.forEach(f)` / `.map(f)` / `.filter(f)` | `unit` / `List<U>` / `List<T>` |
| | `.find(f)` / `.findIndex(f)` / `.some(f)` / `.every(f)` | `T?` / `int` / `bool` / `bool` |
| | `.reduce(f, initial)` | `U` |
| | `.sort(comparator)` | `unit`; mutating; stable |
| `Map<K, V>` | `.size` (property) | `int`, O(1) |
| | `.get(k)` | `V?`; `null` when absent |
| | `.set(k, v)` | `unit`; mutating; replaces or inserts |
| | `.has(k)` | `bool` |
| | `.remove(k)` | `V?`; mutating; the previous value or `null` |
| | `.keys()` / `.values()` / `.entries()` | `List<K>` / `List<V>` / `List<(K, V)>`, insertion order |
| | `Map::fromEntries(entries)` | `Map<K, V>`; static |
| `Set<T>` | `.size` (property) | `int`, O(1) |
| | `.has(v)` | `bool` |
| | `.insert(v)` | `bool`; mutating; `true` when newly added |
| | `.remove(v)` | `bool`; mutating; `true` when removed |
| | `Set::fromList(values)` | `Set<T>`; static |
| `string` | `.len` (property) | `int`; Unicode scalar count |
| | `.byteLen` (property) | `int`; UTF-8 byte count, O(1), no allocation |
| | `.codeAt(i)` | `int`; the scalar at `i` |
| | `.slice(start, end)` | `string`; scalars `start … end − 1` |
| | `.bytes()` | `List<u8>`; the UTF-8 bytes |

Map and Set have **no key subscript and no member entry form**. `m[k]`,
`m[k] = v`, and `m.key` are removed; the key operations are the methods above.
List keeps `xs[i]`/`xs[i] = v`; `string` remains non-indexable.

## Built-in members

- **Inherent and closed.** The members belong to the built-in type. They are
  not `impl` declarations, and a program cannot add an inherent member; a
  trait implementation for a built-in type (`impl Trait for List<T>`) is still
  an ordinary trait method, resolved after the inherent set. The members are
  not prelude operations: `len`, `size`, `get`, `push`, … do not exist as
  unqualified names.
- **Properties versus methods.** `.len`, `.size`, and `.byteLen` are
  read-only properties: `xs.len` has type `int` and is not callable;
  `xs.len = n` is `E0402` (the target is not a place). The others are
  methods, with the ordinary call and method-value rules.
- **Mutation.** `push`, `pop`, `shift`, `unshift`, `splice`, `resize`,
  `truncate`, `clear`, `reverse`, `sort`, `set`, and `remove`/`insert` mutate
  the receiver and therefore require a mutable place (`mut self`); calling one
  on an immutable binding is `E0404`.
- **Callback methods.** The higher-order List members take a synchronous Xulo
  function value and invoke it while the receiver is borrowed. The callback
  receives the element only; `map`/`reduce` introduce a method type parameter
  (`U`) inferred from the callback and initial value. A callback cannot mutate
  the receiver — the ordinary borrow rules reject it (`E0404`/`E0405`) — and a
  callback panic stops the call. The built-in method calling a Xulo function
  value is the one runtime bridge this change adds; callbacks cannot be
  asynchronous.
- **Statics.** `Map::fromEntries` and `Set::fromList` are static members
  reached through `::`, like the existing associated names of `impl` blocks.
  They have no receiver and are never reached through `.`.
- **Resolution order.** Member lookup consults the built-in member set first,
  then inherent `impl` methods, then trait methods in scope. A program cannot
  declare an inherent member for a built-in type, so the first step is closed
  and no user declaration can shadow it. A trait method implemented for a
  built-in type remains a trait method and is considered after the inherent
  set.
- **No shadowing.** Built-in members are not module-scope names. A local
  `let len = …` does not affect `xs.len`; only module-scope intrinsics and
  type names remain reserved.

## Access rules

### Map and Set

- `m.get(k)` returns `V?`: an absent key is `null`, never a runtime failure.
  The key expression is checked against `K`; a mismatch is `E0202`.
- `m.set(k, v)` replaces the entry when `k` is present and inserts it
  otherwise; it returns `unit`. `m.has(k)` answers presence; `m.remove(k)`
  returns the value the key had, or `null` when absent.
- `m.keys()`, `m.values()`, and `m.entries()` return new lists in insertion
  order. The lists are snapshots; mutating the map afterwards does not change
  them.
- `m[k]` and `m[k] = v` are gone; a subscript on a `Map` is `E0215`, the
  same rule that already keeps `string` non-indexable. A member name that is
  not in the Map member set is `E0102`.
- Iteration over a map still yields its keys in insertion order: `for k in m`
  is core syntax and is not a key access. `m.key` is no longer entry lookup;
  the key is `m.get("key")` (or `m["key"]` no longer exists).
- `Map::fromEntries(entries)` builds a map from `(K, V)` pairs in list order;
  the later occurrence of a duplicate key wins, matching map-literal spread.

### Set

- `s.has(v)`, `s.insert(v)`, `s.remove(v)`, and `s.size`; `Set::fromList`
  constructs a set from a list, holding each distinct value once.
- `insert` returns `true` exactly when the value was not present; `remove`
  returns `true` exactly when it was. Both require a mutable place.
- A set still has no iteration order; `for v in s` is permitted and visits the
  elements in an unspecified order, as before.
- A subscript or member entry form never existed on Set; `s[v]` and `s.v`
  are `E0215` and `E0102` respectively.

### List and string

- `xs.len` is the element count, O(1). `xs.push(v)` appends; `xs.pop()` and
  `xs.shift()` remove and return the last or first element, or `null` when the
  list is empty; `xs.unshift(v)` inserts at the front; all are mutating.
- `xs.splice(start, deleteCount, items = [])` removes `deleteCount` elements
  at `start`, inserts the elements of `items` there, and returns the removed
  elements as a new list. `start` is in `0 …= len`; `deleteCount` in
  `0 …= len − start`; outside is a runtime failure. The default `items` is
  the empty list.
- `xs.resize(n, fill)` sets the length to `n` — shrinking from the end or
  appending `fill` — and `xs.truncate(n)` only shrinks (no-op when `n >= len`);
  `xs.clear()` removes all elements. A negative `n` is a runtime failure.
- `xs.slice(start, end?)` returns a new list of `start … end − 1`; a `null`
  `end` means `len`. `start` and `end` are in `0 …= len`, `start <= end`;
  outside is a runtime failure.
- `xs.indexOf(v)`, `xs.lastIndexOf(v)`, and `xs.includes(v)` use `==`; `NaN`
  never equals itself, so it is never found. `xs.join(separator = ",")`
  concatenates the elements' string forms (the `str` domain, `E0212` when the
  element type has none). `xs.reverse()` reverses in place; `xs.entries()`
  yields `(index, element)` pairs in order.
- `xs.forEach(f)`, `xs.map(f)`, `xs.filter(f)`, `xs.find(f)`,
  `xs.findIndex(f)`, `xs.some(f)`, `xs.every(f)`, and
  `xs.reduce(f, initial)` take a synchronous callback and visit the elements
  in order; `find`/`findIndex`/`some`/`every` stop at the first decisive
  element. `map` and `filter` return new lists; `find` is `null` and
  `findIndex` is `-1` when nothing matches; `some` is `false` and `every` is
  `true` for an empty list; `reduce` starts from `initial` and returns it for
  an empty list. The callback receives the element only. `xs.sort(f)` is a
  stable in-place sort; `f(a, b)` is negative/zero/positive for before/equal/
  after; an inconsistent comparator leaves the order unspecified.
- The receiver is borrowed for the duration of every callback member, so a
  callback cannot mutate the same list (a borrow conflict is `E0404`/`E0405`).
  A callback panic stops the call; the callback members do not return
  `Result`.
- `xs[i]` and `xs[i] = v` keep their current meaning and out-of-range
  behavior; a subscript on a list is a place.
- `s.len` counts Unicode scalar values; `s.byteLen` counts UTF-8 bytes and is
  O(1). `s.codeAt(i)`, `s.slice(start, end)`, and `s.bytes()` replace
  `stringCodeAt`, `stringSlice`, and `stringBytes`; their index domains and
  failure behavior are unchanged. `s[i]` remains `E0215`.
- `s.len` may take time proportional to the string's byte length (an
  implementation MAY cache the scalar count); `s.byteLen` is O(1) and
  allocates nothing.

## Alternatives considered

### Keep the prelude functions — the status quo

**Cost.** The four-mechanism surface remains (functions, List subscript, Map
subscript, Map member form), the four `*Size` names stay, and adding a member
API later breaks the same programs anyway. It also leaves Map key access
blocking every future Map member. Rejected: the change is deliberately the
one-time break.

### Members with Map key precedence

**Cost.** Keep `m[key]`/`m.key` and let built-in members win on collision.
`obj.get`, `obj.size`, `obj.keys`, and every future member name silently stop
being JSON keys; the rule is a permanent special case and grows with the
member set. Rejected: the map access form and a member API cannot both stay
unsurprising.

### Remove only the Map member form, keep `m[k]`

**Cost.** `m[k]` remains an access form with a different shape from every
other collection operation, and Map still cannot grow members without
collision. It also leaves the absent-key runtime failure in place. Rejected:
the access split is the problem, not just its dot spelling.

### `.length` / `.size` (JavaScript spelling)

**Cost/benefit.** Strictly matches JavaScript (`Array`/`String` `.length`,
`Map`/`Set` `.size`). `.len` is shorter and pairs with `.byteLen`; the
sequence/container split is already carried by `len`/`size`. JavaScript's
`.length` is also writable and admits holes; a typed `List` has neither, so
`.len` stays read-only and length changes go through
`resize`/`truncate`/`clear`/`splice`. Chosen: `.len`/`.size`, with
`.length`/`.byteLength` the alternative if the longer spelling is preferred.

### A fixed-length `Array<T, N>` in this change

**Cost.** A fixed array needs a type-level length, a literal/layout story,
and FFI/stack semantics; none exists in the current language. It would also
not share the dynamic List members (`push`/`pop`/`splice`/`resize`/`truncate`/
`clear`). Rejected for this change: `List` is the growable sequence; `Array`
is a separate type change.

### Callback members via explicit loops only

**Cost.** Leaving `forEach`/`map`/`filter`/`find`/`some`/`every`/`reduce`/
`sort` to hand-written loops keeps the built-in runtime free of a callback
bridge, but leaves List without the higher-order Array surface this change is
for. Rejected: the bridge is specified here, with synchronous callbacks and
the receiver borrowed for the call.

### `.count()` for Map and Set

**Cost.** Cardinality is stored, not counted; `.count` usually implies an
iteration or a predicate (`iter.count()`), and a method spelling for a stored
O(1) property is a needless second mechanism. Rejected: `.size` is the
standard stored-cardinality name.

### JavaScript method names — `.delete` / `.add`

**Cost.** `delete` is a reserved word in Xulo and cannot be a member name.
`.remove` is therefore forced, and it matches the language's existing
`remove` vocabulary. `.insert` is kept for Set rather than `.add` for the same
consistency; `.add` would be a second spelling for no semantic gain.

### `.set` returns the map, as in JavaScript

**Cost.** Chaining mutators through the receiver is a JavaScript convenience
that buys little in Xulo, where the receiver is already a named place and
methods have explicit `mut self`. Returning `unit` keeps the mutation rules
and the method-value rules simple. Rejected for this change.

### Iterators for `keys`/`values`/`entries`

**Cost.** Xulo has no iterator protocol; the language's `iterator`/`generator`
words are reserved only. Returning `List`s is the existing model
(`mapToEntries` already did) and keeps the change within the current type
system. Rejected until iterators exist.

## Consequences

### Typing and diagnostics

- A member not in the built-in set is `E0102`; a subscript on `Map` or
  `string` is `E0215`; a read-only property as an assignment target is
  `E0402`; a mutation on an immutable binding is `E0404`.
- `m.get` has type `fn(K): V?` with the receiver bound; `m.size` is an
  `int` property, not a function value. `m.size()` is `E0204` (a non-function
  called).
- `m.set(k, v)` checks `k : K` and `v : V`; `m.has`/`m.remove` check `K`;
  `push`/`unshift`/`indexOf`/`lastIndexOf`/`includes` check `T`; `splice`
  checks `int`, `int`, `List<T>`; `slice` checks `int`, `int?`; `resize`
  checks `int`, `T`; `truncate` checks `int`; `pop`/`shift`/`clear`/`reverse`/
  `entries` take no arguments. `join` checks `string` and requires the
  element's string form (`E0212`).
- The callback members check `fn(T): unit`, `fn(T): U`, `fn(T): bool`,
  `fn(U, T): U`, or `fn(T, T): int`; `map` and `reduce` infer their method
  type parameter (`U`) from the callback and initial value. A callback that
  mutably borrows the receiver is `E0404`/`E0405`.
- The static constructors are ordinary `::` calls: `Map::fromEntries` takes
  `List<(K, V)>`, `Set::fromList` takes `List<T>`.

### Runtime failures

- The *Absent map key* condition is deleted from both closed tables: absence
  is `null` from `get`, and `remove` answers `null`. The List condition now
  covers subscripts and the bounds/counts of `splice`/`slice`/`resize`/
  `truncate`; the string index/slice condition remains, spelled with
  `s.codeAt`/`s.slice` and `s.len`. A callback's own panic is an ordinary
  panic and does not add a new closed-table row.
- `map`/`filter` allocate new lists; `sort` is stable and O(n log n) in time.
  The callback bridge runs synchronously; a callback panic aborts the call
  and the partially built result is discarded by the collector.

### Names

- The prelude keeps its types, `ToString`, namespaces, and intrinsics; the
  collection and string operation sections leave. `mapGet`, `listPush`,
  `stringSize`, and the rest are no longer names at all, and their module-scope
  reservation disappears.
- `.len`, `.size`, `.byteLen`, and the method names are member names, not
  module-scope identifiers; a program may still use them for locals or fields.

### Documentation and implementation

- The type chapters own the member tables; `path-and-access` and
  `checking-rules` own the access and typing rules; `prelude.md` shrinks to
  the non-collection prelude and points at the type chapters.
- The implementation provides the built-in member sets, the property reads,
  and the static constructors; Map/Set subscript resolution is removed from
  the checker. The parent standard-library specification's examples that
  call `listSize` or use Map subscripts are updated after acceptance.
