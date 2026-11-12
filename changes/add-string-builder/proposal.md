# Add StringBuilder

**Status:** draft · **Date:** 2026-11-12 · **Affects:**
`types/composite-types.md`, `types/README.md`, `types/primitive-types.md`,
`builtins/prelude.md`, `builtins/intrinsic-functions.md`, `names.md`,
`memory-and-runtime.md`, `expressions/operators.md`,
`type-system/checking-rules.md`, `type-system/coercion.md`, `README.md`,
`changes/README.md`

## Summary

String concatenation with `+` creates a new immutable `string` and copies the
whole accumulated prefix on every operation, so assembling text one fragment
at a time in a loop is quadratic. This change adds the prelude type
`StringBuilder` — a mutable, growable UTF-8 text buffer with reference
semantics — and a small member set:

> `StringBuilder::new()`, `.append(s: string)`, `.clear()`, `.build()`,
> `.len`, `.byte_len`

```xulo
let mut sb = StringBuilder::new()
sb.append("count=")
sb.append(str(n))
sb.append(" items")
let text = sb.build()          // a string: "count=3 items"
```

The builder is a reference value like `List`, so binding or passing it shares
one buffer; `copy` makes an independent one. Its `==` compares the current
text. It has no literal, no subscript, no iteration, no `Display`
implementation, and no implicit conversion to or from `string`: `.build()`
materializes the result. No grammar, AST, diagnostic, or runtime-failure row
is added; the deltas in [`specs/`](specs/) are the text the chapters gain once
the change is accepted.

## Motivation

### Concatenating in a loop is quadratic

`+` is the only way to concatenate two strings, and it returns a new
`string` each time:

```xulo
let mut text = ""
for x in xs {
  text = text + str(x)      // copies the whole prefix on every iteration
}
```

With n fragments this allocates and copies 1 + 2 + … + n characters — Θ(n²)
work for a job that should be linear. The cost is not incidental: building a
diagnostic message, a generated source file, or a protocol line is a loop of
appends, and the immutable-string rule makes the obvious loop the slow one.

### The linear workaround is not a builder

`List<string>` with `.join(separator)` produces the final text in linear time,
but it is a different shape: the program must retain one list element per
fragment until the very end, cannot observe or clear the current text, and
must keep the list plumbing beside the text logic. `StringBuilder` is the
direct form of the same operation — append text to one buffer, read its
current length, clear it, materialize it — and it is the form a reader expects
when the word "builder" appears. Its `.append` is amortized O(n) in the
appended scalars, so a loop of appends is linear in the total text.

### A library type cannot finish the job

The specification has one-way byte access (`s.bytes()`) but no constructor
from bytes or scalar values. A standard-library `StringBuilder` over
`List<u8>` could accumulate bytes, yet it could not return a `string` without
an unspecified conversion; a checked constructor would need UTF-8 validation
and a `Result`-typed API, and an unchecked one would break the guarantee that
a `string` is valid UTF-8. Making the builder a built-in lets `.build()` be
total: every append accepts a `string`, so the buffer is valid UTF-8 at every
point, and the result needs no failure channel.

### The type slots into existing rules

`StringBuilder` is the reference-semantics, content-bearing shape the language
already has for `List`, `Map`, and `Set`: shared on binding, deep-copied by
`copy`, compared structurally, mutated only through a mutable place, and
materialized with an ordinary member. It needs no operator, no literal, and
no control-flow form. That keeps the change to type, member, prelude, name,
memory, equality, coercion, and rendering text.

## Scope

**In scope.** The built-in type `StringBuilder`; its static constructor
`StringBuilder::new()`; the members `.append`, `.clear`, `.build`, `.len`, and
`.byte_len`; its reference semantics, UTF-8 invariant, content equality, and
amortized append cost; its place in the prelude, reserved names, built-in type
kinds, value/reference and `copy` rules, equality and coercion rules, and the
`print` rendering table.

**Out of scope.** Capacity and reservation members (`with_capacity`,
`.capacity`, `.reserve`); appending values other than `string` (use a
template or `str`); appending a builder directly (use `.build()`); a `Display`
implementation or derived `.to_string()`; indexing, iteration, or slicing a
builder; search, replace, trim, or split; UTF-8 validation APIs; an FFI
representation; hashing; conversion from `string` to `StringBuilder` other
than `new()` followed by `append`; and any new diagnostic code or
runtime-failure row.

## Compatibility

- No existing expression changes meaning: `StringBuilder` did not exist, so no
  program could have named the type, constructed it, or called its members.
- A module-scope declaration named `StringBuilder` becomes a reserved-name
  error (`E0106`), exactly as the addition of `Array` did. Inner scopes may
  still shadow the name under the ordinary rule.
- `sb[i]`, `for x in sb`, `str(sb)`, `${sb}`, and `sb == "x"` are rejected by
  rules that already exist (`E0215`, `E0213`, `E0212`, `E0211`); the change
  only places the builder on their existing lists.
- `print(sb)` gains a rendering: the buffer's current text, as the `string`
  row renders it. This is the one new observable behavior, and it is confined
  to a type no existing program could produce.

## Non-goals

- No capacity API. `List` exposes no capacity, and the specification leaves
  allocation strategy to the implementation; a builder may grow however it
  chooses as long as appends remain amortized linear.
- No generic `append` over `Display`. A `T: Display` bound does not cover the
  base types without an implementation, and rendering to a temporary string
  before appending is already the `str`/template path.
- No `StringBuilder` literal. `StringBuilder::new()` is the only constructor;
  an initial text is written as `new()` followed by `append`.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] The change adds no grammar production, AST node, diagnostic code, or
      runtime-failure row.
- [x] All affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md).
