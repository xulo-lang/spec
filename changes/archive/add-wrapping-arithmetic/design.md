# Design: wrapping and bit operations in `Math`

## The decision

Eight new `Math` members, in one new `### Integer functions` table
between Basic functions and Formatting:

| Member | Signature | Meaning |
|--------|-----------|---------|
| `Math::wrappingAdd` | `fn(a: int, b: int): int` | `a + b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrappingSub` | `fn(a: int, b: int): int` | `a − b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::wrappingMul` | `fn(a: int, b: int): int` | `a × b` modulo 2⁶⁴, reinterpreted as two's complement |
| `Math::popCount` | `fn(x: int): int` | the number of 1 bits in `x` |
| `Math::countLeadingZeros` | `fn(x: int): int` | the number of leading 0 bits — 64 when `x` is 0 |
| `Math::countTrailingZeros` | `fn(x: int): int` | the number of trailing 0 bits — 64 when `x` is 0 |
| `Math::rotateLeft` | `fn(x: int, amount: u64): int` | `x` rotated left by `amount` modulo 64 |
| `Math::rotateRight` | `fn(x: int, amount: u64): int` | `x` rotated right by `amount` modulo 64 |

The `wrapping*` results are the two's-complement re-interpretation of the
value modulo 2⁶⁴ — the unique result in `int`'s range — and never trap.
The bit functions read and write the same 64-bit representation;
`rotateLeft`/`rotateRight` take the amount modulo 64, so an amount of 0
or 64 is the identity and no amount is out of domain.

## Alternatives considered

### Change the operator default — wrap silently, Go-style

**Cost.** Deletes the trap-everywhere policy that the Overflow rule, the
two failure tables, and the `E0210`/trap split all exist to enforce;
`x - 1` at the minimum would become a silent wrap in the one place the
spec promised a stop. Rejected outright — the wrapping functions are the
escape hatch precisely so the default need not move.

### Operator forms — `+&`, `*%`, or a `wrapping` block

**Cost.** New operator tokens or a new statement form: a grammar change
with precedence and tooling consequences, for a need that is
quantitatively small (bit code) and can be met with calls. Function
calls keep the surface closed and the parse table untouched. Rejected
for v1; an operator spelling can be added later without breaking call
sites.

### Fixed-bit spelling — `wrappingAdd<T>(a: T, b: T): T`

Generic over the integer types, so `u32` hashes work directly. **Cost.**
`Math`'s existing numeric vocabulary is `int`/`float`/`number`-typed;
a generic `T: int-family` bound would be the first trait-bounded
intrinsic in the namespace, pulling the generic checker into prelude
semantics. Hash and bitset code in the bootstrap is `int`-native, and a
fixed-bit user can wrap, add, and unwrap only through casts the language
does not have — so that user's real path is `int` anyway. Rejected; the
`usize`/fixed-bit story can grow as one change when a consumer appears.

### Wrapping functions without the bit functions

**Cost.** The table, the index rows, and the review are the expensive
part; `popCount` and the two counts are total, trivially specified, and
wanted by the same bitset code that wants wrapping. Splitting them into
a second change pays the fixed cost twice for no scoping benefit.
Rejected: one change, one table.

### The chosen set, `int`-only, in `Math`

**Why.** It is the smallest closed table that unblocks the bootstrap's
hash and bit paths; it reuses the namespace the spec already designates
for numeric vocabulary; and every function is total or explicitly
wrapping — nothing joins the trap tables. The checkable deliverable is
precise: eight closed intrinsics with total signatures and exact
modular/bit semantics, no grammar change, no diagnostic.

## Consequences

### Runtime and typing

- The three `wrapping*` functions never produce a runtime failure; the
  Overflow bullet gains one sentence saying so, so the two rules cannot
  be read against each other.
- The bit functions are total: `countLeadingZeros(0) = 64`,
  `countTrailingZeros(0) = 64`, `rotateLeft(x, 64) = x`.
- `amount` is `u64` on the rotates — the same width family as
  `Time::sleep(ms: u64)` — and is taken modulo 64, so no amount is
  out of range and no new failure condition exists.

### Diagnostics

| Code | Change |
|------|--------|
| `E0210` | unchanged — operator overflow is untouched; the new bullet only says the intrinsics are outside the rule |
| (new codes) | none |

### Grammar

None: no production, token, or precedence changes.

### Complete signature index

Eight rows, alphabetical: `countLeadingZeros`/`countTrailingZeros` after
`Math::cos`; `popCount` between `Math::PI` and `Math::pow`;
`rotateLeft`/`rotateRight` between `Math::pow` and `Math::round`;
`wrappingAdd`/`wrappingSub`/`wrappingMul` after `Math::trunc`.

### Parity

The change adds one table, two bullets, and eight index rows of reading;
in exchange it adds eight closed intrinsics with total, exactly
specified semantics — a new checkable property (wrapped arithmetic is
available and never traps) that the Overflow rule cross-references. No
new syntax appears, so the gate's syntax clause is untouched.
