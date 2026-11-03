# Composite Types — Delta: Snake-Case Names

> **Delta** — replaces the `List<T>` inherent-member list and its two
> following paragraphs, the `Array<T, N>` shared-member line, the `Map`
> static constructor, and the `Set::fromList` reference in
> `types/composite-types.md`.

## List<T> (inherent members)

- The type has these inherent members:
  - `.len` — a read-only property of type `int`, the number of elements,
    O(1);
  - `.push(v: T)` — append `v`; mutating; returns `unit`;
  - `.pop()` — remove and return the last element, or `null` when the list is
    empty; mutating; type `T?`;
  - `.shift()` — remove and return the first element, or `null` when the list
    is empty; mutating; type `T?`;
  - `.unshift(v: T)` — insert `v` at the front; mutating; returns `unit`;
  - `.splice(start: int, delete_count: int, items: List<T> = [])` — remove
    `delete_count` elements at `start`, insert the elements of `items` there,
    and return the removed elements as a new `List<T>` in order; mutating.
    `start` is in `0 …= len`, and `delete_count` in `0 …= len − start`;
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
  - `.index_of(v: T)` — the first index whose element is `== v`, or `-1`; type
    `int`;
  - `.last_index_of(v: T)` — the last such index, or `-1`; type `int`;
  - `.includes(v: T)` — whether some element is `== v`; type `bool`;
  - `.join(separator: string = ",")` — the elements' string forms joined by
    `separator`, with `""` for an empty list; type `string`. The element type
    MUST be a base type or implement `Display` (the `str` domain); otherwise
    the call is a compile-time error (`E0212`);
  - `.reverse()` — reverse the list in place; mutating; returns `unit`;
  - `.entries()` — a `List<(int, T)>` of `(index, element)` pairs, in order;
  - `.for_each(f: fn(T): unit)` — call `f` on each element in order; returns
    `unit`;
  - `.map<U>(f: fn(T): U)` — a new `List<U>` of the callback results, in
    order;
  - `.filter(f: fn(T): bool)` — a new `List<T>` of the elements for which `f`
    returns `true`, in order;
  - `.find(f: fn(T): bool)` — the first element for which `f` is `true`, or
    `null`; type `T?`;
  - `.find_index(f: fn(T): bool)` — the index of that element, or `-1`; type
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
  `index_of`, `last_index_of`, and `includes` require `T` to admit `==`; `NaN`
  never equals itself, so it is never found.

## List<T> (callback members paragraph)

- The callback members take a synchronous Xulo function value — `fn(T): U`,
  `fn(T): bool`, `fn(U, T): U`, or `fn(T, T): int` as the member states. The
  callback receives the element only, not its index (use `.entries()` for
  index/value pairs). `for_each`/`map`/`filter`/`reduce` visit every element;
  `find`/`find_index`/`some`/`every` stop at the first decisive element. The
  receiver is borrowed for the duration of the call, so a callback cannot
  mutate it; an ordinary borrow conflict is reported (`E0404`/`E0405`). A
  callback that panics stops the call; the callback members do not return
  `Result`. `sort` order is unspecified for a comparator that does not define
  a consistent ordering.

## Array<T, N> (shared members)

- The read, callback, and sort members are the List members — `.slice`,
  `.index_of`, `.last_index_of`, `.includes`, `.join`, `.entries`,
  `.reverse`, `.sort`, `.for_each`, `.map`, `.filter`, `.find`,
  `.find_index`, `.some`, `.every`, `.reduce` — with these differences:

## Map<K, V> (static constructor)

  - `Map::from_entries(entries)` — a static constructor from a
    `List<(K, V)>`, in list order; a repeated key keeps the later entry.

## Set<T> (static constructor)

- The language defines no set literal: the brace syntax belongs to `Map` — `{}` is its empty literal — so a set is constructed with the static member `Set::from_list(values)`.
