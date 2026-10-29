# Change Collection Members

**Status:** accepted · **Date:** 2026-10-27 · **Affects:**
`types/composite-types.md`, `types/primitive-types.md`,
`expressions/path-and-access.md`, `expressions/control-flow.md`,
`functions.md`,
`type-system/checking-rules.md`, `type-system/errors.md`,
`builtins/prelude.md`,
`builtins/README.md`, `builtins/intrinsic-functions.md`, `names.md`,
`statements/let-and-assignment.md`, `testing.md`, `error-handling.md`,
`memory-and-runtime.md`

## Summary

The built-in collections and `string` expose their operations as inherent
members, and Map/Set lose direct key access. The surface becomes:

| Type | Members |
|------|---------|
| `List<T>` | `.len` (read-only), `.push`, `.pop`, `.shift`, `.unshift`, `.splice`, `.resize`, `.truncate`, `.clear`, `.slice`, `.indexOf`, `.lastIndexOf`, `.includes`, `.join`, `.reverse`, `.entries`, `.forEach`, `.map`, `.filter`, `.find`, `.findIndex`, `.some`, `.every`, `.reduce`, `.sort`; `xs[i]` and `xs[i] = v` stay |
| `Map<K, V>` | `.size` (read-only), `.get(k): V?`, `.set(k, v)`, `.has(k)`, `.remove(k): V?`, `.keys()`, `.values()`, `.entries()`; `Map::fromEntries(entries)` |
| `Set<T>` | `.size` (read-only), `.has(v)`, `.insert(v): bool`, `.remove(v): bool`; `Set::fromList(values)` |
| `string` | `.len` (read-only; Unicode scalars), `.byteLen` (read-only; UTF-8 bytes), `.codeAt(i)`, `.slice(start, end)`, `.bytes()` |

Map and Set have **no key subscript and no member entry form**: `m[k]`,
`m[k] = v`, and `m.key` are gone. A key is read with `get` — which returns
`null` when absent — written with `set`, tested with `has`, and removed with
`remove`. The prelude functions that used to carry these operations
(`mapGet`, `listPush`, `stringSize`, `setInsert`, …) are removed; the
remaining prelude is types, protocols, namespaces, and intrinsics.

`List` is the **growable sequence** (dynamic array), so it takes the dynamic
operations of a JavaScript `Array` — `push`/`pop`/`shift`/`unshift`, `splice`,
`resize`/`truncate`/`clear`, `slice`, `reverse`, and `entries` — together with
`indexOf`/`lastIndexOf`/`includes`/`join` and the callback members
`forEach`/`map`/`filter`/`find`/`findIndex`/`some`/`every`/`reduce`/`sort`.
A fixed-length `Array<T, N>` is a different type and is **out of scope** for
this change; its length would be type-level and it would have none of the
dynamic members.

## Motivation

### Higher-order methods need a callback bridge

The callback members are the remaining Array surface: `forEach`/`map`/`filter`
/`find`/`findIndex`/`some`/`every`/`reduce` and `sort(comparator)` make a
built-in List method invoke a Xulo function value. The language already has
first-class functions and closures; this change specifies that bridge — the
callback is synchronous, receives the element only, and runs while the
receiver is borrowed (so it cannot mutate the same list). The alternative,
hand-written loops, leaves the common Array surface out and is rejected.

### One operation, one member spelling

The old surface mixed four mechanisms for one family of operations: prelude
functions for most (`listPush`, `mapGet`, `setInsert`, `stringSize`), one
subscript for List and Map, and one member form for string-keyed maps. A
caller had to know both the receiver type and which mechanism its operation
used. The standard-library specification paid for it: writing examples it
reached for `xs.length`, a spelling that exists nowhere in the language.

The member spelling is the one every value already offers through `.`:
`xs.len`, `counts.get("a")`, `tags.insert("x")`, `s.slice(1, 3)`. It is also
the JavaScript model for `Map`/`Set` — `map.get`, `map.set`, `map.has`,
`map.size`, `set.has`, `set.size` — with Xulo's `.insert`/`.remove`
spellings, which is the model that motivated this change.

### Map key access and members cannot coexist

`Map<string, V>` currently reads an entry with `m.key` and `m[k]`. Any built-in
member added to Map then collides with a key of the same name: `m.get` would
be the entry `"get"` rather than a method, and `m.size` the entry `"size"`
rather than the cardinality. There is no type-directed escape — the entry's
type is `V`, the member's type is fixed — and a precedence rule would
silently reserve every member name as a map key. The only coherent choices
are to keep Map function-only (the status quo) or to remove direct key
access. This change takes the second: Map and Set are method-accessed, as
their JavaScript counterparts are. `get` also improves the failure shape:
absence is `null`, not a runtime error.

### Length for sequences, size for containers

`List` and `string` are sequences: `.len` is their extent, and a string's
`.byteLen` is its UTF-8 byte extent. `Map` and `Set` are containers: `.size`
is their cardinality. This is the JavaScript and Java split (`Array`/`String`
length, `Map`/`Set` size); it keeps the short `.len` familiar for sequences
without calling a keyed container "long".

### The byte count is missing and is not free today

`stringSize` counts scalars. The UTF-8 byte count is what a protocol needs
(`Content-Length` above all), and today the only way to get it is
`listSize(stringBytes(s))` — a full byte list allocated just to be measured.
`.byteLen` answers it without the allocation.

## Scope

**In scope.** Built-in members on the four types; the List dynamic members,
its callback bridge, and the extended List bounds/length failure row; the
removal of Map/Set key
subscripts and the Map member form; the removal of the prelude collection and
string functions; the closed, inherent member-set mechanism; read-only
`.len`/`.size`/`.byteLen`; `get` returning `null`; the deletion of the
*Absent map key* runtime-failure row; the updates to every chapter that names
the old operations or the old access forms.

**Out of scope.** A fixed-length `Array<T, N>` (a separate type change:
layout, literal syntax, and FFI); iterators (`keys`/`values`/`entries` return
`List`s); asynchronous callbacks and `flatMap`/`flat`-style methods; Map `set`
returning the map for chaining; changing Set's unspecified order or Map's
insertion order; changing map literals; writable `.len`/`.length` (`.len = n`
remains invalid; length changes through `resize`/`truncate`/`clear`/`splice`);
user extension of the built-in member sets.

## Compatibility

- **Source-breaking.** `m[k]`, `m[k] = v`, and `m.key` no longer exist; every
  old prelude operation name is gone; the prelude `listRemove` is replaced by
  `splice`/`pop`, and `List` gains the dynamic members. The migration is
  mechanical and happens before 1.0.
- `get` replaces the trapping key read: absence is `null`, so the *Absent map
  key* runtime failure is deleted. List subscripts, `splice`/`slice` bounds,
  and negative `resize`/`truncate` lengths still trap.
- List keeps Xulo shapes where JavaScript differs: `pop`/`shift` return
  `null` on an empty list, `push`/`unshift`/`reverse`/`clear` return `unit`,
  `splice` takes a `List<T>` of items instead of varargs, bounds are
  non-negative (`0 …= len`), and `indexOf`/`includes` use `==` (so `NaN` is
  not found). `.len` is read-only: a typed `List` has no holes and no
  `undefined`, so the writable `length` of a JavaScript array does not carry
  over.
- The callback members are synchronous and receive the element only, not its
  index or the list; `reduce` requires an initial value (there is no
  no-initial form), and `sort` requires a comparator — there is no default
  string sort — and is stable. `forEach` returns `unit`; `map`/`filter` create
  new lists; `find` returns `null`; `findIndex` returns `-1`; `some` is
  `false` and `every` is `true` for an empty list.
- `delete` is a reserved word and cannot name a member; Map and Set use
  `.remove`, matching the language's existing `remove` vocabulary. Set keeps
  `.insert`.
- No grammar change: `[]` remains an operator (List); `.` and `::` already
  reach members. The built-in members are not prelude names and do not reserve
  module-scope identifiers.
- Map literals and map key iteration (`for k in m`) stay; Set iteration keeps
  its unspecified order.
