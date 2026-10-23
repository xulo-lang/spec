# Change Wrapping Arithmetic

**Status:** accepted · **Date:** 2026-10-23 · **Affects:**
`builtins/intrinsic-functions.md`, `types/primitive-types.md`

## Summary

Integer overflow is closed: a constant overflow is `E0210`, a
non-constant one a runtime trap, and there is no sanctioned way to
compute a wrapped result. Hash functions, checksums, bit mixing, and
every port of C's unsigned arithmetic need exactly that — today a
self-hosted implementer must either avoid the operation or invent an
intrinsic. This change adds a closed set of eight `Math` functions:

> `Math::wrappingAdd`, `Math::wrappingSub`, `Math::wrappingMul` —
> two's-complement results modulo 2⁶⁴, never trapping; and
> `Math::popCount`, `Math::countLeadingZeros`,
> `Math::countTrailingZeros`, `Math::rotateLeft`,
> `Math::rotateRight` — bit counting and rotation on the 64-bit
> representation.

The operator overflow rule is untouched — the wrapping functions are the
escape hatch, not a change of defaults — and `types/primitive-types.md`
gains one sentence pointing the Overflow rule at them. The deltas in
[`specs/`](specs/) are the text the chapters gain once the change is
accepted.

## Motivation

### The trap policy is right for arithmetic and wrong for bit mixing

`hash(3) * 31 + 1` overflowing is a bug in ordinary arithmetic code and
should stop the program — that is the policy the language rightly keeps.
In a hash accumulator or a CRC, overflow is the *definition* of the
operation: the same `*` and `+` are used, and the caller wants the low 64
bits, not a panic. Without a specified wrapped form, every self-hosted
hash implementation either calls FFI, hand-rolls modular arithmetic with
divisions (slow, and still needs care at 2⁶⁴), or trips the trap at
runtime. The language owes bit code a direct path.

### Self-hosting needs it on day one, not library day

The first Xulo program is a compiler; the first thing a compiler hashes
is names and source spans. The collection prelude already carries the
operations types need but syntax does not; `Math` already carries
`floor`/`trunc` as numeric vocabulary. Wrapped arithmetic is the same
class of vocabulary, closed and tiny, and it belongs with the other
intrinsics rather than behind an FFI the bootstrap should not need.

### The bit functions come with the package

`popCount`, `countLeadingZeros`, and `countTrailingZeros` are the other
three primitives every bitset, encoder, and size class needs, and they
cannot overflow or trap — they are total functions on the 64-bit
representation. Shipping them with the wrapping trio means one table, one
bullet list, one index pass, instead of three future trips through the
same review.

## Scope

**In scope.** The eight `Math` functions and their semantics (mod-2⁶⁴
wrapping; bit operations on the 64-bit two's-complement representation;
rotate amounts taken modulo 64); the two Math-section bullets; the eight
rows of the complete signature index; the one-sentence cross-reference in
the Overflow rule of `types/primitive-types.md`.

**Out of scope.** Any change to operator overflow (trapping stays);
operator forms (`+&` and the like); wrapping on fixed-bit types other
than `int`; wrapping division or remainder; checked (Result-returning)
variants; a `usize`-typed spelling of any of the eight.

## Compatibility

No conforming program can regress: eight new prelude-visible names join
the closed `Math` namespace; a program cannot have been calling them
since they did not exist. The overflow rule's verdict on `a + b` is
unchanged, so every existing program keeps its meaning. The only
observable effect on existing code is that the names are now taken by
the namespace — `Math` members are reached through `Math::`, and no
module-scope declaration is involved.

## Non-goals

- No wrapping negation: `Math::wrappingSub(0, x)` covers it.
- No `checkedAdd` and friends: `Result`-typed arithmetic is standard-
  library material and was not requested by the bootstrap path.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] All three affected passages (Math tables, Math bullet list, index,
      Overflow bullet) are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md).
