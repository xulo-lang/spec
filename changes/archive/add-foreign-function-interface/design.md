# Add Foreign Function Interface — Design

**Status:** accepted · **Date:** 2026-10-21
**Change ID:** add-foreign-function-interface

This file is the record of the design discussion: every fork that was
considered, the alternatives with their costs, the choice made, and why. It
carries no normative text — the normative deltas are under
[`specs/`](specs/).

## 1. Problem

Xulo's strongest claim is that it defines no undefined behavior
([`memory-and-runtime.md`](../../memory-and-runtime.md)): for every program
that passes the static checks, meaning is fixed. The introduction also
declares no FFI contract a non-goal. Those two positions held each other in
balance only as long as nothing crossed the boundary. A native presentation
target needs platform C APIs; without a specified `extern`, bindings invent
conventions outside the specification where no checker can police them.

So the question was never *whether a boundary can exist*, but **where to draw
it so that the zero-UB claim survives the drawing**.

## 2. The layering: how much pointer does v1 need?

The first real fork. Three layers were compared:

| Layer | What crosses | New UB surface | What becomes declarable |
|-------|--------------|----------------|--------------------------|
| **L0** — copy boundary | value copies only | **zero** | `abs`, `sqrt`, `getpid`, integer-handle protocols |
| **L1** — opaque handle | copies + a non-dereferenceable `Ptr` token | **zero** (all dereference happens in foreign code, outside the specification) | `malloc`/`free` pairing, `FILE*`-style handles, window handles |
| **L2** — readable buffer | borrowed `Slice<u8>` (ptr+len), C strings | bounded but real: needs either dereference intrinsics or a GC pinning obligation | `printf` with real text, pixel-buffer upload, `read(fd, buf, n)` |

**Alternatives considered:**

- *L0 only (no pointer type at all).* Cleanest guarantee — only value copies
  cross — but it cannot express the change's own motivating examples:
  `malloc`/`free` is un-writable without a way to hold the returned handle,
  and most real C APIs pass at least one pointer-shaped argument. Rejected as
  too weak: an FFI that cannot say "handle" is not an FFI.
- *L0 + L1 + L2 in one step.* Full expressive power now. Rejected: L2 forces
  one of two expensive foundations — dereference intrinsics (reintroducing
  UB-class reads into the language, directly against the parity principle,
  which requires syntax to *buy* a guarantee rather than spend one) or a
  pinning obligation on the garbage collector (an edit to the root of
  [`memory-and-runtime.md`](../../memory-and-runtime.md), whose collection
  mechanism is deliberately left free). Both are real designs deserving their
  own proposal; neither can be rushed inside v1.
- *Full C pointers (`Ptr<T>` with dereference and arithmetic).* Rejected for
  three compounding reasons: it contradicts the zero-UB claim at its widest
  point; it has no foundation in the memory model, because the collection
  mechanism is unspecified and may move objects, so a pointer into the Xulo
  heap has no defined validity; and it reopens the door the introduction
  already keeps shut — "memory layout, inline assembly, and bare-metal
  control are outside the language's ambitions".

**Chosen: L0 + L1.** The insight that settled it: *the philosophical line is
not "is there a pointer type" but "can the language side dereference one"*.
An opaque token creates no new UB surface — the type system's whole job is
that you cannot mistake a handle for an integer and that null-testing is
explicit — while every dereference stays in foreign code, beyond the
specification by construction.

The guarantee this buys is stated as the chapter's creed in
[`specs/ffi.md`](specs/ffi.md) (bolded, at the top):

> **What crosses the boundary is value copies and opaque tokens: Xulo never
> dereferences a foreign pointer, foreign code can never reach the Xulo heap,
> and what foreign code does itself lies outside this specification.**

## 3. The null model for `Ptr`

| Alternative | Cost |
|-------------|------|
| **A. Non-null `Ptr`; nullable written `Ptr?`** (chosen) | none measurable: `Ptr?` is exactly `Ptr \| null`, so `??`, `?.`, `is`, `match`, and null-comparison narrowing all apply unchanged; no new intrinsic is needed |
| **B. `Ptr` admits `null` itself, as in C** | breaks the language-wide rule that there is no implicit nullability ([`types/primitive-types.md`](../../types/primitive-types.md)); forces a `ptrIsNull` intrinsic into the prelude where the ordinary optional vocabulary already does the job |

Chosen **A**. Prior art: Rust spells the same choice as `NonNull<T>` +
`Option<NonNull<T>>`; Zig as `?*T`. Xulo needs neither trick — its optional
type constructor already models a nullable handle exactly. The design rule
"check `Ptr?` before use where `Ptr` is required" falls out of the existing
`E0201` rule for optionals with no FFI-specific text.

Consequence for signatures: `malloc` returns `Ptr?` and `free` takes `Ptr?`
— faithful to C, where `free(NULL)` is a no-op, and the null check at the
call site is ordinary code:

```xulo
let p = cMalloc(64) ?? panic("out of memory")   // p: Ptr
```

## 4. Surface syntax: keyword block vs attribute

| Alternative | Cost |
|-------------|------|
| `#Extern(C)` attribute on ordinary `fn`s | no new keyword, reuses the closed attribute registry; but imports have no home — every symbol is a separate top-level declaration — and the attribute registry's argument form (`AttributeArg ::= Identifier`) has no place for a string ABI tag without extending the grammar anyway |
| **`extern` block for imports + `extern` modifier for exports** (chosen) | one new keyword (58 → 61 together with `usize`/`isize`); group form reads like `mod x { … }`, which a reader already knows |
| Hybrid: block for imports, attribute for exports | two mechanisms to learn for one feature; rejected as surface without benefit |

Chosen: the keyword forms, which match the shape the discussion started from:

```xulo
extern "C" {
  fn malloc(size: usize): Ptr? as cMalloc
}

pub extern "C" fn add(a: i32, b: i32): i32 { a + b }
```

Grammar consequences: `ExternBlock`, `ExternItem`, `ExternParamList`,
`ExternParam`, `AbiTag` are new productions; the export form is a new
alternative of `Declaration` rather than an extension of `FnDecl`, because an
extern export admits no `async`, no generic parameters, no `where` clause, no
attribute list, and no receiver — positions `FnDecl` already has. Writing the
export as its own production makes those absences *syntactic*, so they cost
no diagnostic code (the same technique `TestDecl` uses).

## 5. ABI tag spelling: string or identifier

The discussion's original tension: the module-system change had just removed
string specifiers from the language ("no string literal appears anywhere in
the module system"), and this proposal brings one back.

**Alternatives:**

- *Bare identifier, `extern C { … }`.* Consistent with the module system's
  de-stringification; extensible to a path (`sysv_amd64`).
- *String literal, `extern "C" { … }`* (chosen). Reads as what it is — an
  **opaque foreign name**, not a path in Xulo's own namespace. A module
  specifier names *Xulo's* structure, so it became `::` paths; an ABI tag
  names something that is none of Xulo's business, and a string states that
  directly. Strings also scale better across the registry: `"C"`, `"system"`,
  later `"wasm"` — and `"c++"` (§7) would be an awkward bare identifier
  anyway. Familiar to anyone who has seen Rust or Zig.

Chosen: **string literal**, with the cost paid honestly — `AbiTag ::= StringLiteral`
in the grammar, and *legality is a checking rule, not a grammatical one*,
exactly as the attribute registry does it. The grammar admits any string
literal; the checker validates it against the closed registry (`E0902`).

## 6. The registry: what v1 admits, and what it never will

| Tag | Verdict | Reasoning |
|-----|---------|-----------|
| `"C"` | registered | the only ABI with a stable cross-platform convention; per-platform variation (SysV AMD64, AAPCS64, MS x64) dissolves into "the target platform's C ABI" |
| `"system"` | registered | the portable spelling of the platform default convention; covers 32-bit Windows `stdcall` system APIs at zero cost. Its interpretation rides on the same implementation-defined entry as `"C"` |
| `"wasm"` | not registered; reserved for a later change | there is no WASM compilation target yet; when there is, the registry design already accommodates a per-ABI type-mapping table |
| `"c++"` | not registered; expected never to register in-language | there is no single C++ ABI (Itanium and MSVC name mangling are incompatible), templates cannot be declared as extern signatures, and exception propagation across the boundary is a separate language problem. The ecosystem consensus is that C++ interop lives at the library layer (Rust's `cxx`, Swift's interop via tooling), not in the language |
| `"go"` | does not exist as a thing | Go exposes no stable ABI for external languages to bind to; `//export` *exports the C ABI*. Calling Go is calling `"C"` |

Unknown tag → `E0902`, with a note listing the registered set (the `E0111`
pattern). Adding a tag in a future version is a minor change that breaks no
program.

## 7. Types at the boundary

**Chosen whitelist (closed):** `bool`, `i8`…`u64`, `f32`, `f64`, `usize`,
`isize`, `unit`, `Ptr`, `Ptr?` — and nothing else.

Alternatives weighed:

- *Include `int` and `float` (the user-facing layer).* Rejected: `int` is
  "whatever the platform calls int" in spirit but not in law — the formal
  fixed-bit layer exists precisely so the boundary says `i32`/`i64` and not
  `int`. Excluding the user-facing layer costs nothing (C's `abs` takes
  `int32_t`, not a 64-bit `int`, so the annotation would have to be written
  anyway) and buys explicitness where explicitness pays.
- *Admit any `T?`.* Rejected: only `Ptr?` is C-representable (a nullable
  pointer); `i32?` would need a niche that the C ABI does not define. The
  rule is therefore "the only optional type in an extern signature is `Ptr?`"
  rather than a general optional rule.
- *Admit `struct` with a layout attribute.* Deferred to v2: Xulo structs are
  nominal GC-managed records with no defined layout; making one ABI-safe
  requires a layout annotation whose design (field order, padding, packed vs
  aligned, interaction with the collector) is a proposal of its own.
- *Admit function types for callbacks.* Deferred to v2 (§10).
- *Admit `string`.* Rejected for v1 on two independent grounds: `string` is
  GC-managed UTF-8 with no NUL terminator guarantee, and lifetime rules for a
  borrowed C string do not exist yet. Library-level wrappers remain possible
  later; the language need not own them in v1.

Everything not on the list — `string`, `List`/`Map`/`Set`, `struct`, `enum`,
tuples, `View`, `Task`, `Result`, `unknown`, `number`, unions, function types
— is `E0901`. `unit` is allowed as the return type (or omitted) and rejected
as a parameter type under the same code — `E0901`: C has no `void`
parameter to map it to.

**Diagnostic split.** The division of labor follows the repository's own
convention — grammar decides what parses, checking decides what is true:

- *Syntax (no code):* bodies absent in import items, `async`/generics/
  `where`/attributes/receivers absent from both extern forms, parameter modes
  (`mut`, `shared`) and default values absent from `ExternParam`, export form
  requiring `pub`.
- *Checking (new codes):* `E0901` non-ABI-safe type, `E0902` unregistered ABI
  tag, `E0903` extern import referenced outside direct call position.
- *Inherited codes:* name collisions `E0103`/`E0104`, prelude-reserved names
  `E0106`, and a type application on the non-generic `Ptr` (`Ptr<u8>`) under
  the existing type-application rule `E0230`.

Three codes, not more: v1's semantic surface is small by design.

## 8. `usize` / `isize`: hard or contextual keyword

A reviewer suggested making them **contextual keywords** so that
`let usize = 10` stays writable. Rejected:

- every other type in the numeric family — `int`, `u64`, `f32` — is a hard
  keyword that may not be used as a name in any position; a lone exception
  inside one table is a lexical inconsistency a reader will trip over;
- Xulo's contextual-keyword mechanism has exactly one established use, the
  `@State`-family words directly after `@`; extending it to type names opens a
  second mechanism for a benefit (one variable name) that cannot carry it;
- the specification does not forbid `let usize = 10` *by accident* — it
  forbids `let int = 10` by the same rule, deliberately.

The same reasoning confirms `extern` as a hard keyword. The cost is honest
and noted: any code already using `extern` as an identifier loses it. The
specification is a draft (README §2), and the precedent set by
`lowercase-primitive-keywords` — a keyword-set change of far larger radius —
was accepted through this same workflow.

`usize`/`isize` widths are **implementation-defined** (documented per target)
and therefore join the closed list in
[`memory-and-runtime.md`](../../memory-and-runtime.md); they map to C's
`size_t` and `ptrdiff_t`. Their arithmetic follows the fixed-bit rules:
closed under same-type operands, and mixing with any other numeric type is
`E0201`.

## 9. Names, aliasing, and visibility

- **`as` direction.** `fn printf(fmt: Ptr): i32 as cPrintf` — the declared
  name is the foreign symbol, the name after `as` is the **local binding**,
  exactly parallel to `use path::x as y` where the right side is local. One
  rule, two features.
- **Export symbol names.** `pub extern "C" fn add` exports the symbol `add` —
  the Xulo name *is* the symbol name. A symbol that collides with a Xulo
  keyword (a C function named `type` or `match`) cannot be written as an
  identifier; supporting it needs a string symbol name. Deferred to v2 as a
  deliberate narrowness: the collision is rare, and inventing string syntax
  for a rare case now would widen the surface for nothing.
- **Block-level `pub`.** `pub extern "C" { … }` exports every item in the
  block; items carry no individual `pub`. The block shares one ABI, so
  visibility shares it too. An export *form* is only `pub extern "C" fn` —
  linkage without export has no observable meaning in v1, so the grammar does
  not admit a private extern function declaration.
- **Calling.** Extern imports are callable in direct call position only;
  storing one in a binding or passing it as a value is `E0903`. Rationale:
  an Xulo value of function type `fn(…): R` carries no ABI, so a call through
  it would use Xulo's own convention on a foreign symbol — correct by luck,
  wrong by construction. The fix is a typed `extern "C" fn(…): R` function
  pointer (v2), not a silent assumption.
- **Everything else is inherited unchanged:** left-to-right argument
  evaluation, named arguments, optional-parameter omission (so `cFree()` with
  a `Ptr?` parameter passes `null`, faithful to `free(NULL)`), and argument
  borrowing for the call's duration. No new call rules — a sentence in
  [`specs/ffi.md`](specs/ffi.md) says so rather than restating them.

## 10. The carve-out in the zero-UB claim

The claim "this specification defines no undefined behavior" cannot survive
contact with foreign code untouched. The design choice is *where* to cut:

- **Rejected: a general `unsafe` region.** The language has no unsafe
  construct and no user-facing unsafe code; importing one for FFI would be a
  far larger cultural change than the feature warrants.
- **Chosen: a boundary carve-out keyed to the extern construct itself.** The
  amendment to [`memory-and-runtime.md`](../../memory-and-runtime.md) keeps
  the claim's force for every construct of the language and states that what
  a foreign callee does *during an extern call* lies outside the
  specification — while the declaration side (ABI-safe types, registered tag,
  unique symbol names, direct-call-only) remains fully checked. Rust draws
  the same line at `unsafe extern`; Xulo draws it at `extern`, with the
  narrower claim that the Xulo side of the boundary contributes no
  undefinedness of its own: it passes copies and tokens.

Reentrancy — a foreign callee reaching an exported symbol — is possible only
through exports and follows ordinary call rules; it needs no special text,
and is recorded here for completeness.

## 11. Where operations live

No `ptrRead`, `ptrWrite`, `ptrOffset`, `ptrNull`, or `ptrIsNull` intrinsics
exist in v1 — not as a trimming, but because L1 needs none: null-testing is
the optional type's vocabulary (§3), and there is nothing else to do with a
token besides compare it and hand it back. `==`/`!=` on `Ptr` (address
equality) are the only operators. Ordering comparisons are not defined; the
prelude's numeric conversions reject `Ptr` under `E0201` like any
non-numeric.

What remains is storage, not interpretation. A `Ptr` is an ordinary value
on the Xulo side: `let`-bound, stored in a struct field or a `List<Ptr>`,
passed into and out of ordinary functions and closures (the
`body: fn(Ptr): i32` pattern in [`specs/ffi.md`](specs/ffi.md)), with the
collector treating the token as an opaque word. The whole design turns on
this split: operations that *interpret* a pointer value — read, write,
offset, cast — are the forbidden family; operations that *move the token*
are ordinary language operations that need no FFI rule. Neither widens the
boundary: a struct of `Ptr` fields is still not ABI-safe in an extern
signature, and an imported symbol is not a value to store — only to call
(`E0903`).

The known trap from Rust's history is recorded as a permanent rule: **a `Ptr`
never converts to or from an integer.** Rust spent years on integer↔pointer
provenance (`expose_provenance` and its predecessors); the escape hatch is
what made that swamp necessary. Xulo's `Ptr` is opaque with no conversion, so
the question cannot arise. This is a "never", listed as such in
[`specs/ffi.md`](specs/ffi.md).

## 12. Example idioms (and the trap worth documenting)

Four sanctioned ways to handle a `Ptr?`, all inherited from `T?`:

1. `if p != null { … }` — comparison narrows the binding inside the branch;
2. `p ?? panic("…")` — coalescing with an adaptable `panic` for
   allocate-or-die (`panic` may be given any type the context requires);
3. `match p { Ptr q => …, null => … }` — when both arms produce a value;
4. `p ?? other` — when a real fallback handle exists.

The trap: **narrowing is branch-local.** The specification narrows bindings
inside `if` branches and `match` arms only — there is no flow-sensitive
narrowing after an early exit — so the Rust-shaped

```xulo
let p = cMalloc(64)
if p == null { return null }
cFree(p)             // error[E0201]: p is still Ptr? here
```

is rejected, and the accepted shapes rebind (`?? panic`, `match`) or enclose
the use in the branch. This is an existing language rule, not an FFI rule;
it surfaces in FFI code often enough that [`specs/ffi.md`](specs/ffi.md)
demonstrates it beside the four idioms.

## 13. Scope boundary: what the language does not do

Linking and loading — which library holds `malloc`, when it is opened, how a
manifest maps a package to a path — stay outside the language, as README
§1.1 already assigns the manifest to tooling. The extern declaration names a
symbol and an ABI; resolution is the host's business. This keeps the feature
inside the specification's existing jurisdiction instead of dragging the
package model in with it.

## 14. v2 debt list

Recorded in [`specs/ffi.md`](specs/ffi.md) as a closed "not in scope" list
(normative absence) and here with the open questions each one carries:

| Debt | Open question |
|------|---------------|
| `Slice<u8>` borrowed buffers + C-string bridging | the GC pinning obligation: whose? — a borrowed slice needs "storage not moved or reclaimed for the call's duration", an obligation on the runtime that touches the free collection mechanism |
| `Ptr` ↔ `Slice<u8>` conversion | must be split into two directions: Xulo heap → slice (in-boundary, needs pinning) vs foreign `Ptr` + length → slice (out-of-boundary, Xulo reads foreign memory). A plausible v2 answer is to **forbid** both implicit conversions so that foreign-owned memory (`Ptr`) and Xulo-owned buffers (`Slice`) never blur — the blur is where use-after-free hides |
| `extern "C" fn(…): R` function pointers | enables callbacks; the ABI must live on the function *type*, not only on the declaration |
| string symbol names for exports | unblocks exporting under a name that is a Xulo keyword |
| `"wasm"` ABI tag | needs a WASM compilation target and a per-ABI type-mapping table |
| `#Layout(…)`-style struct annotation | field order, padding, and interaction with the collector |
