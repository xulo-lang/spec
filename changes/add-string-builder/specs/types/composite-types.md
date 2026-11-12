# Composite Types — Delta: StringBuilder

> **Delta** — replaces the chapter introduction and adds the
> `## StringBuilder` section after `## Set<T>` in
> `types/composite-types.md`; the rest of the chapter is unchanged.

Replace the chapter introduction with:

Composite types are built from other types. They fall into three groups: the
built-in collections `List<T>`, `Array<T, N>`, `Map<K, V>`, and `Set<T>`,
together with the built-in text builder `StringBuilder`; the record types —
nominal `struct` records and positional tuples; and the constructors that
combine types — `T?`, `T | U`, and `T & U`. This chapter specifies the values
each constructor admits, how values of those types are written, and the rules
by which values move between them.

Insert after `## Set<T>`:

## StringBuilder

`StringBuilder` is a built-in mutable, growable UTF-8 text buffer. It is a
**reference** type: binding, assignment, argument passing, and return share
one buffer, and a write through any mutable place naming it is visible through
every other binding ([`../memory-and-runtime.md`](../memory-and-runtime.md)).

```xulo
let mut sb = StringBuilder::new()
sb.append("count=")
sb.append(str(3))
sb.append(" items")
let text = sb.build()          // "count=3 items"
```

- `StringBuilder::new()` is the only constructor: a static member, reached
  through `::`, that returns a new, empty builder. There is no literal form.
- The type has these inherent members:
  - `.len` — a read-only property of type `int`, the number of Unicode scalar
    values currently in the buffer, O(1); the builder maintains this scalar
    count as text is appended and cleared;
  - `.byte_len` — a read-only property of type `int`, the number of UTF-8
    bytes currently in the buffer, O(1); the builder maintains this byte count
    as text is appended and cleared;
  - `.append(s: string)` — append the Unicode scalar values of `s`; mutating;
    returns `unit`. Appending `n` scalar values runs in amortized O(n) time,
    so a loop that appends to one builder is linear in the total text, not
    quadratic;
  - `.clear()` — remove all text; mutating; returns `unit`;
  - `.build()` — a new `string` holding the buffer's current text; the builder
    is unchanged, and later appends do not affect the returned string.
- The mutating members `.append` and `.clear` require the receiver to be a
  mutable place ([`../memory-and-runtime.md`](../memory-and-runtime.md)); on an
  immutable receiver the call is `E0404`. `.build`, `.len`, and `.byte_len`
  read the receiver and require no mutability.
- `.append` accepts only a `string`. A value of another type is rendered first
  with a template or `str`, exactly as for `+`
  ([`primitive-types.md`](primitive-types.md)); the builder does not convert
  its argument. Appending one builder's text is `sb.append(other.build())`.
- Everything appended is already a valid UTF-8 string, so the buffer always
  holds a valid UTF-8 sequence; `build` cannot fail and needs no `Result`.
- `sb[i]` is not an operation: a subscript on a `StringBuilder` is a
  compile-time error (`E0215`), and `for x in sb` is `E0213`. A builder is
  written and read through its members, not indexed or iterated.
- `==` and `!=` are defined for two `StringBuilder` operands and compare the
  current text by content, like `string`. A `StringBuilder` and a `string`
  have no common type, so `sb == "x"` is `E0211` — compare
  `sb.build() == "x"`.
- `StringBuilder` is not a collection of elements and has no `Display`
  implementation: `str(sb)` and `${sb}` are compile-time errors (`E0212`);
  materialize the text with `.build()`. `print(sb)` renders the current text
  by the output table
  ([`../builtins/intrinsic-functions.md`](../builtins/intrinsic-functions.md)).
- A `StringBuilder` has no FFI representation and is not hashable, and no
  implicit conversion exists between `StringBuilder` and `string` in either
  direction.
