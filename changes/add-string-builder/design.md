# Design: StringBuilder

## The decision

`StringBuilder` is a built-in, non-generic **reference** type in the prelude.
Its value is a mutable, growable UTF-8 text buffer; the current text is always
a valid UTF-8 sequence because only `string` values are appended.

| Member | Signature | Meaning |
|--------|-----------|---------|
| `StringBuilder::new` | `fn(): StringBuilder` | a new, empty builder; the only constructor, reached through `::` |
| `.len` | read-only `int` | the number of Unicode scalar values currently in the buffer, O(1); maintained as a running scalar count |
| `.byte_len` | read-only `int` | the number of UTF-8 bytes currently in the buffer, O(1); maintained as a running byte count |
| `.append` | `fn(s: string): unit` | append the scalar values of `s`; mutating; amortized O(n) for n appended scalars |
| `.clear` | `fn(): unit` | remove all text; mutating |
| `.build` | `fn(): string` | a new `string` holding the current text; the builder is unchanged |

The type is shared on binding and assignment, like `List`. `.append` and
`.clear` require a mutable place; `.len`, `.byte_len`, and `.build` need only
a readable receiver. `copy sb` produces an independent buffer. `sb1 == sb2`
compares the current text by content. `sb == text` is `E0211` because the two
operands have no common type — compare `sb.build() == text`.

The builder is not a text value: it has no `Display` implementation, so
`str(sb)` and `${sb}` are `E0212`, and `.build()` is the explicit
materialization. `print(sb)` renders the current text through the output
table's new `StringBuilder` row. There is no literal, subscript, iteration,
coercion, hashing, or FFI representation.

## Alternatives considered

### Keep `+` as the only append operation

**Cost.** Every `s = s + fragment` copies the accumulated prefix. A loop is
Θ(n²); the type that exists to make text assembly linear is missing. Rejected.

### `List<string>` plus `.join`

**Cost.** This is the current linear workaround and it works, but it is not a
buffer: every fragment must stay in a list until the end, the program cannot
read or clear the current text without another join, and the list is separate
state from the text being assembled. A builder is the direct expression of
"append, inspect, clear, materialize" and does not retain a parallel list.
Rejected as the primary API; it remains available.

### A standard-library type

**Cost.** The language has no `string` constructor from bytes or scalar
values. A library builder over `List<u8>` could append, but `.build()` could
not produce a `string` without either an unchecked conversion — breaking the
UTF-8 invariant — or a checked one returning `Result`, which would make every
build fallible. A built-in builder appends only valid strings, so `.build()` is
total. Rejected.

### A byte buffer over `List<u8>`

**Cost.** It exposes byte offsets, UTF-8 encoding, and validation to every
caller, and still cannot materialize a `string` without the missing
constructor. It solves the wrong level of the problem. Rejected.

### Value semantics

**Cost.** A value-semantic builder would copy its buffer on binding,
assignment, argument passing, and return — the operations a builder is used to
avoid — and would make a helper that appends to a parameter require a
`mut` value copy or a return value. It is also the opposite of the reference
precedent set by `List`, `Map`, and `Set`. Rejected.

### `Display` / derived `.to_string()` for finalization

**Cost.** It would make `str(sb)`, `${sb}`, `List.join` on builders, and
`.to_string()` silently materialize a string, giving a mutable buffer a text
form that changes under the reader. It also requires a special built-in
`Display` implementation, while the existing built-ins with content render
through the output table, not through `Display`. An explicit `.build()`
states when the snapshot is taken. Rejected.

### Generic `append<T: Display>`

**Cost.** A `T: Display` bound requires a visible `impl`, which the base types
do not have; covering both base types and `Display` would add a special bound
spelling for one method. Rendering to a temporary string with `str` or a
template is already specified and keeps `.append` one total signature.
Rejected.

### Capacity members

**Cost.** `with_capacity`, `.capacity`, and `.reserve` expose allocation
policy, which the memory model deliberately leaves to the implementation.
`List` exposes none of them. The amortized append guarantee is the useful
contract; a capacity hint is an optimization the implementation may already
perform. Rejected for this change.

### Fluent or consuming finalization

**Cost.** Returning the builder from `.append` would allow chaining but
departs from every mutating member in the language, which returns `unit`.
Consuming the builder in `.build()` would make a second build impossible and
add ownership complexity for no benefit; the builder stays usable and the
returned `string` is an independent value. Rejected.

### A builder with no `.clear` or length

**Cost.** The smallest possible append/build pair cannot be reused for the
next message without allocating another builder, and a caller cannot ask how
much text it has assembled. `.len`, `.byte_len`, and `.clear` complete the
buffer abstraction at four small operations. Rejected.

### Built-in reference type with the chosen member set — chosen

**Why.** It makes the quadratic loop linear, fits the existing reference /
copy / equality / mutable-place rules without a new concept, keeps the
UTF-8 invariant total, and adds no syntax. The deliverable is checkable: one
type, one constructor, five members, one rendering row, and a set of list
membership edits; no new diagnostic or runtime failure.

## Consequences

### Prelude and names

- `StringBuilder` joins the built-in types of the prelude and the reserved
  names (`E0106` at module scope).
- `StringBuilder::new` is a built-in static member, reached through `::` like
  `Map::from_entries`; the bare name `new` is in scope nowhere.
- The built-in-members paragraph names the builder beside the collections and
  `string`.

### Typing

- The type is non-generic and nominal: `StringBuilder` is equal only to
  itself, and there is no subtyping or coercion to or from `string`.
- `.len` and `.byte_len` are read-only properties, not places.
- `.append` takes exactly one `string`; a value of another type is rendered
  first with a template or `str`. There is no overloading.
- `.append` and `.clear` are mutating members and follow the ordinary
  mutable-place and borrow rules for a `mut self` receiver (`E0404` on an
  immutable receiver).
- `==` and `!=` compare the current text; a builder and a `string` have no
  common type (`E0211`).
- `sb[i]` is a subscript on a non-`List`/non-`Array` type (`E0215`);
  `for x in sb` is a non-iterable (`E0213`); `str(sb)` and `${sb}` are
  `E0212`, with `.build()` as the replacement.

### Runtime

- `StringBuilder` is a reference value: binding and assignment share one
  buffer, a write through any mutable place naming it is visible through every
  other binding, and `copy` deep-copies the buffer.
- The buffer holds only appended `string` values, so its text is always valid
  UTF-8 and `.build()` cannot fail.
- `.append` is amortized O(n) in the appended scalars; `.build` is O(total
  bytes); `.len` and `.byte_len` are O(1). The buffer maintains a running
  scalar count and a running byte count as text is appended and cleared, so
  neither property walks the text; that running count is what supports the
  O(1) claim. No allocation strategy or capacity is observable.
- No new runtime-failure row: there is no index, no bound, and no decoding
  step to fail.

### Diagnostics

| Code | Change |
|------|--------|
| `E0106` | unchanged — `StringBuilder` joins the reserved names via the prelude rule |
| `E0102` | unchanged — a member outside `new`/`len`/`byte_len`/`append`/`clear`/`build` is reported as a missing member |
| `E0211` | unchanged — `sb == "x"` has no common type |
| `E0212` | unchanged — `str(sb)` and `${sb}` have no string form; `.build()` materializes |
| `E0213` | unchanged — a builder is not iterable |
| `E0215` | unchanged — a builder is not subscriptable |
| `E0404` | unchanged — `.append`/`.clear` need a mutable receiver |

No diagnostic is registered or retired.

### Grammar

None. `StringBuilder::new()` is an ordinary path call, and `.append`, `.clear`,
`.build`, `.len`, and `.byte_len` are ordinary member accesses. No production,
token, precedence, or AST node changes.

### Parity

The change adds one built-in type, one static constructor, five members, one
rendering row, and a set of list-membership edits; in exchange it removes the
quadratic text-assembly shape without adding syntax. It is a Layer 2
(typing/memory) and Layer 3 (conformance examples) change; the grammar layer
is untouched.
