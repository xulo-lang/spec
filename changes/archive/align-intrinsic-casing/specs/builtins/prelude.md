# Prelude — Delta: Intrinsic Casing

> **Delta** — renames the multi-word intrinsics in the **Built-in
> protocols** table, the **Map operations**, **Set operations**, and **List
> operations** sections of `builtins/prelude.md`. Signatures and meanings are
> otherwise unchanged.

## Built-in protocols — table row and bullet, replaced

| Protocol | Method | Required by |
|----------|--------|-------------|
| `ToString` | `fn toString(self): String` | interpolation in a template literal, and the intrinsic `str` |

- A type implements `ToString` with `impl ToString for T`, whose `toString`
  returns the string form of the receiver
  ([`../types/traits.md`](../types/traits.md)).

(The other bullets of the section are unchanged.)

## Prelude operations — example block, replaced

The example block after the "prelude operation" paragraph becomes:

```xulo
let mut counts: Map<String, Int> = { "a": 1, "b": 2 }
print(mapSize(counts))              // 2
print(mapHasKey(counts, "a"))      // true
print(mapGet(counts, "z"))          // null: absent, and no failure
let previous = mapRemove(counts, "a")   // counts loses "a"; previous is 1

let tags: Set<String> = setFromList(["a", "b", "a"])
print(setSize(tags))                // 2: the duplicate collapsed
let added = setInsert(tags, "c")
print(setContains(tags, "c"))       // true

let mut xs = ["a", "b"]
listPush(xs, "c")                   // xs is ["a", "b", "c"]
let removed = listRemove(xs, 0)     // "a"; xs is ["b", "c"]
for e in enumerate(xs) {
  print(`${e.0}=${e.1}`)             // 0=b, then 1=c
}

let user = { name: "lyy", age: 30 }  // a Map<String, String | Int>
print(user["name"])                  // lyy
```

## Map operations — table and bullets, replaced

| Signature | Meaning |
|-----------|---------|
| `mapSize<K, V>(m: Map<K, V>): Int` | the number of entries in `m` |
| `mapHasKey<K, V>(m: Map<K, V>, k: K): Boolean` | `true` when `k` has a binding in `m` |
| `mapGet<K, V>(m: Map<K, V>, k: K): V?` | the value `k` is bound to, or `null` when `k` has no binding |
| `mapRemove<K, V>(m: mut Map<K, V>, k: K): V?` | removes the binding for `k` and returns the value it had, or `null` when `k` was absent |
| `mapToEntries<K, V>(m: Map<K, V>): List<(K, V)>` | one `(key, value)` tuple per entry, in insertion order (bulk conversion) |
| `mapFromEntries<K, V>(entries: List<(K, V)>): Map<K, V>` | builds a map from `(key, value)` tuples, in list order; a repeated key keeps the later entry |

- Reading (`m[k]`), writing (`m[k] = v`), and iterating the keys (`for k in m`)
  are core syntax and need no prelude function
  ([`../types/composite-types.md`](../types/composite-types.md)).
- `mapGet` is the lookup that cannot fail: reading `m[k]` for a key with no
  binding is a runtime failure, while `mapGet(m, k)` answers `null`
  ([`../error-handling.md`](../error-handling.md)).
- `mapRemove` mutates its first argument, so that argument MUST be a mutable
  place — a `let mut` binding or a `mut` parameter; passing anything else is a
  compile-time error (`E0401`).

## Set operations — table and bullets, replaced

| Signature | Meaning |
|-----------|---------|
| `setFromList<T>(values: List<T>): Set<T>` | constructs a set holding each distinct element of `values` exactly once |
| `setInsert<T>(s: mut Set<T>, v: T): Boolean` | adds `v` to `s`; returns `true` when `v` was not already present |
| `setRemove<T>(s: mut Set<T>, v: T): Boolean` | removes `v` from `s`; returns `true` when `v` was present |
| `setContains<T>(s: Set<T>, v: T): Boolean` | `true` when `v` is a member of `s` |
| `setSize<T>(s: Set<T>): Int` | the number of elements in `s` |

- `setInsert` and `setRemove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `mapRemove`.
- A set has no specified order: neither `setFromList` nor any other
  operation defines the order of the elements, and programs MUST NOT depend on
  one ([`../types/composite-types.md`](../types/composite-types.md)).

## List operations — table and bullets, replaced

| Signature | Meaning |
|-----------|---------|
| `listSize<T>(xs: List<T>): Int` | the number of elements in `xs` |
| `listPush<T>(xs: mut List<T>, v: T): Unit` | appends `v` to the end of `xs` |
| `listRemove<T>(xs: mut List<T>, i: Int): T?` | removes the element at index `i`, shifting later elements down, and returns it; `null` when `i` is negative or at or past the end |
| `enumerate<T>(xs: List<T>): List<(Int, T)>` | one `(index, element)` tuple per element, in order |

- `listPush` and `listRemove` mutate their first argument, which MUST
  therefore be a mutable place, exactly as for `mapRemove`.
- `listRemove` is the removal that cannot fail: the subscript `xs[i]` for an
  out-of-range index is a runtime failure, while `listRemove(xs, i)` answers
  `null` and leaves `xs` unchanged.

(The introductory prose of each section is unchanged.)
