# Primitive Types — Delta: Pointer-Width Integers and `Ptr`

> **Delta** — adds a **Pointer-width integers** subsection to
> `types/primitive-types.md`, extends the C/Rust/WASM mapping table and the
> **Reserved Type Names** list, and adds a cross-reference for `Ptr`.

The new subsection, inserted immediately after the fixed-bit C-mapping table,
before **Literal defaulting and coercion**:

### Pointer-width integers

| Type | Width | Range |
|------|-------|-------|
| `usize` | `W` bits, unsigned — the platform's pointer width, implementation-defined | 0 to 2^W − 1 |
| `isize` | `W` bits, signed — the same `W` | −2^(W−1) to 2^(W−1) − 1 |

- `usize` and `isize` are formal-layer types whose width is the width of the
  platform's pointer: sizes, capacities, indices, and C's `size_t` and
  `ptrdiff_t`.
- The value of `W` is implementation-defined and an implementation MUST
  document it (32 or 64 in practice), together with the ABI document it comes
  from ([`../memory-and-runtime.md`](../memory-and-runtime.md)).
- A literal used where `usize` or `isize` is required follows the fixed-bit
  literal rule below: `let n: usize = 8` has type `usize`, and a literal out
  of range is a compile-time error (`E0209`).
- Arithmetic follows the fixed-bit promotion rule below: `usize + usize` is
  `usize`; mixing either type with any other numeric type — fixed-bit of
  another width, `int`, `number` — is a compile-time error (`E0201`), since
  a signed and an unsigned quantity of implementation-defined width have no
  defined mixed result. There is no coercion between `usize` and `isize`, or
  between either and any fixed-bit type ([`../type-system/coercion.md`](../type-system/coercion.md)).

The fixed-bit C-mapping table gains two rows:

| Xulo | C / Rust | WASM |
|------|----------|------|
| `usize` | `size_t` / `usize` | `i32` (32-bit) or `i64` (64-bit), the pointer width |
| `isize` | `ptrdiff_t` / `isize` | `i32` (32-bit) or `i64` (64-bit), the pointer width |

The reserved list becomes: `List`, `Map`, `Set`, `Task`, `Range`, `Result`,
`View`, `Error`, `Ptr`.

The closing paragraph gains: "`Ptr` is specified in
[`../ffi.md`](../ffi.md); it names an opaque foreign handle, not a value
type, and is reserved for that reason."

No other section of this chapter changes: `Ptr` is not a primitive value
type, does not appear in the formal-layer table, and participates in no
literal defaulting or arithmetic rule — its rules live in
[`../ffi.md`](../ffi.md).
