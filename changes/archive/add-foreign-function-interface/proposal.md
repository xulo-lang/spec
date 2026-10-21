# Add Foreign Function Interface

**Status:** accepted · **Date:** 2026-10-21
**Change ID:** add-foreign-function-interface

## What

This change gives Xulo a foreign-function interface: a way to **declare**
functions that live outside the language and to **export** Xulo functions under
a foreign ABI. It adds:

- the `extern` declaration forms — an import block
  `extern "C" { fn …; }` and an export `pub extern "C" fn … { … }` — with a
  **string ABI tag drawn from a closed, checker-validated registry**
  (`"C"`, `"system"` in v1);
- two pointer-width integer types, `usize` and `isize`, for sizes and C's
  `size_t`/`ptrdiff_t` (both are hard keywords;
  [`design.md`](design.md) §8 records why they are not contextual);
- one new type, `Ptr` — an **opaque, non-null, non-dereferenceable** handle —
  and its ordinary optional form `Ptr?` for nullable handles;
- a closed list of **ABI-safe types** that may appear in an `extern` signature
  (`bool`, the fixed-bit types, `usize`/`isize`, `unit`, `Ptr`, `Ptr?`);
- the new diagnostic family `E0901`–`E0999` (foreign boundary);
- a **carve-out** in the "no undefined behavior" claim of
  [`memory-and-runtime.md`](../../memory-and-runtime.md): what foreign code
  does during an extern call lies outside this specification, while the
  declaration side stays fully checked.

It also removes the corresponding non-goal from
[`introduction.md`](../../introduction.md) — the bullet "No foreign-function
interface contract" — and updates the document map and terminology table of
[`README.md`](../../README.md), the keyword set of
[`lexical-structure.md`](../../lexical-structure.md) (58 → 61), and the
grammar of [`grammar.md`](../../grammar.md).

## Why

Three arguments, in order of weight:

1. **The language is already FFI-shaped; it just has no door.** The formal
   fixed-bit layer exists, per [`introduction.md`](../../introduction.md), "for
   ABI and FFI precision", and
   [`types/primitive-types.md`](../../types/primitive-types.md) maps it
   directly to C, Rust, and WASM types. A program can lay a value out
   compatibly with a C ABI and then has no way to *call* across it. Every
   native presentation target — a native window above all — needs platform
   services (clipboard, notifications, dialogs, screen metrics) that are C
   APIs; without `extern`, "binding Xulo to another language's code is a
   library and host concern" names a library and host that cannot exist.

2. **The surface costs almost nothing because the boundary is a copy
   boundary.** In the design chosen here (see
   [`design.md`](design.md)), only value copies and opaque tokens cross: Xulo
   never dereferences a foreign pointer, foreign code can never reach the Xulo
   heap, and no new UB surface is created on the Xulo side. The language buys
   that guarantee with one block form, one modifier, two keywords, and one
   type — while the checker keeps every promise it makes elsewhere: signatures
   are checked against a closed whitelist, ABI strings against a closed
   registry.

3. **The alternative was tried and does not hold.** The status quo — no
   `extern` form at all — leaves native-target programs with no specified way
   to reach their host, which means each binding invents its own convention
   outside the specification, where the checker cannot police it. A specified
   `extern` moves the one place foreign code touches Xulo *into* the checker's
   view.

### Examples of what becomes possible

```xulo
extern "C" {
  fn abs(x: i32): i32
  fn getpid(): i32
  fn malloc(size: usize): Ptr? as cMalloc
  fn free(p: Ptr?) as cFree
}

pub extern "C" fn add(a: i32, b: i32): i32 { a + b }   // ok
```

And four ways to guard a nullable handle — the chapter and
[`design.md`](design.md) §12 show the same four:

```xulo
let p = cMalloc(64)
if p != null { cFree(p) }                 // 1: the guard narrows inside the branch

fn mustAllocate(n: usize): Ptr {
  cMalloc(n) ?? panic("out of memory")    // 2: allocate-or-die; binds a `Ptr`
}

match p {                                 // 3: both arms produce a value
  Ptr q => { cFree(q); "freed" }
  null => "empty"
}

let handle = cMalloc(8) ?? backup         // 4: a real fallback handle
```

Narrowing is branch-local, so an early-return guard does not carry — that
trap, and the rebinds that avoid it, are in [`design.md`](design.md) §12.

## Scope

**In scope:**

- import blocks and export declarations, their grammar, visibility, and
  call rules (direct calls only);
- the ABI registry with `"C"` and `"system"`; how a registered tag is
  interpreted (the target platform's C ABI) is implementation-defined but MUST
  be documented;
- the ABI-safe type whitelist, including the rules that `string`, collections,
  `struct`, `enum`, tuples, `View`, `Task`, `Result`, function types, `async`,
  generics, receivers, parameter modes, and default parameter values do not
  appear in an `extern` signature;
- `Ptr` / `Ptr?`: construction (only as an extern result or pass-through),
  `==`/`!=`, ordinary optional handling; never dereference, arithmetic, or
  conversion to and from integers — in this version or any later one. On the
  Xulo side of the boundary a `Ptr` is an ordinary value: `let`-bound,
  storable in a struct field or a `List<Ptr>`, passable into and out of an
  ordinary Xulo function — a wrapper wraps the call, never holds the imported
  symbol itself (`E0903`). The ordinariness stops at the boundary: a struct,
  even one whose fields are all `Ptr`, is not ABI-safe (`E0901`), and only
  `Ptr` and `Ptr?` cross;
- the boundary carve-out in [`memory-and-runtime.md`](../../memory-and-runtime.md)
  and the corresponding implementation-defined entry (which platform ABI
  document `"C"` and `"system"` denote, hence the width of `usize`/`isize`);
- diagnostics `E0901` (non-ABI-safe type in an extern signature), `E0902`
  (unregistered ABI tag), `E0903` (extern import referenced outside a direct
  call).

**Out of scope (stated here so their absence is deliberate, not an oversight):**

- **Linking, loading, and symbol resolution** — which library a symbol comes
  from, when it is opened, and how a manifest names it. The language specifies
  the declaration; the host resolves it (README §1.1 already assigns the
  manifest to tooling).
- **Strings, buffers, and collections across the boundary** — `string` is
  GC-managed UTF-8 with no NUL guarantee, and no borrowed-slice type exists.
  A borrowed `Slice<u8>` with a pinning obligation is a v2 design.
- **`Ptr` ↔ `Slice<u8>` conversion** — distinct from the borrowed buffer
  above, and likewise deferred: Xulo heap → slice needs a GC pinning
  obligation for the call's duration; foreign `Ptr` plus length → slice
  makes Xulo read memory it does not own. A plausible v2 answer is to forbid
  both implicit conversions so that foreign-owned memory and Xulo-owned
  buffers never blur; the open question is recorded in
  [`design.md`](design.md) (§14), beside the borrowed-buffer debt.
- **Callbacks**: no function-pointer type crosses the boundary, so foreign
  code cannot call back into Xulo in this version.
- **C++ and other runtime ABIs** — `"c++"` is not registered and is not
  expected to be; C++ interop is a library concern (as in Rust, via `cxx`-style
  tooling). There is no "Go ABI" to register: `//export` from Go *is* the C
  ABI.
- **Struct layout, `as`-style casts of handles, and exporting a symbol whose
  name collides with a Xulo keyword** — all deferred, with reasons, in
  [`design.md`](design.md).
- **Wide (vector) calling conventions** and architecture-specific tags beyond
  `"system"`; the registry is closed in v1 and grows only by later change.

## Relationship to existing chapters

| Chapter | Effect |
|---------|--------|
| [`introduction.md`](../../introduction.md) | non-goal bullet removed and replaced by a narrower one |
| [`lexical-structure.md`](../../lexical-structure.md) | `extern`, `usize`, `isize` added to the keyword set (58 → 61) |
| [`grammar.md`](../../grammar.md) | `Keyword`, `Declaration`, the ambiguity note, new `Extern*` productions |
| [`ast.md`](../../ast.md) | `DeclarationNode` gains `ExternBlockDeclaration` and `ExternFunctionDeclaration`; three node interfaces |
| [`functions.md`](../../functions.md) | one cross-reference bullet in Definition; Visibility unchanged |
| [`types/primitive-types.md`](../../types/primitive-types.md) | `usize`/`isize` in the formal layer; `Ptr` cross-reference |
| [`memory-and-runtime.md`](../../memory-and-runtime.md) | foreign-boundary carve-out; fourth implementation-defined entry |
| [`type-system/errors.md`](../../type-system/errors.md) | range `E0901`–`E0999` and its first three codes |
| [`README.md`](../../README.md) | document map row for `ffi.md`; terminology entries |
| **`ffi.md` (new)** | the chapter itself, as `specs/ffi.md` |
