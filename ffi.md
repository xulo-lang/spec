# Foreign Function Interface

A `.xulo` program reaches code written outside the language through **extern
declarations**: an import block declares foreign symbols, and an export
declaration publishes a Xulo function under a foreign ABI. This chapter
defines those two forms, the ABI registry, the types that may cross the
boundary, and the one type — `Ptr` — that names a foreign address without
granting access to it.

**What crosses the boundary is value copies and opaque tokens: Xulo never
dereferences a foreign pointer, foreign code can never reach the Xulo heap,
and what foreign code does itself lies outside this specification.**

## Declarations

### Import block

```xulo
extern "C" {
  fn abs(x: i32): i32
  fn getpid(): i32
  fn malloc(size: usize): Ptr? as cMalloc
  fn free(p: Ptr?) as cFree
}

// ok
```

- The form is `extern AbiTag { … }` at module level or inside an inline `mod`
  body; the declarations it holds are **signature-only**. The block's
  `AbiTag` is a `StringLiteral` and MUST name a registered ABI
  ([ABI tags](#abi-tags)); an unregistered tag is a compile-time error
  (`E0902`).
- An item has the form `fn name(params) [: Type] [as local] ;`. The declared
  `name` is the **foreign symbol**; the optional `as local` binds that symbol
  in this module under `local` instead, exactly as `use path::x as y` renames
  the local binding only. An item with no `as` binds its declared name.
- An item carries **no body**. It also admits no `async` marker, no generic
  parameters, no `where` clause, no attribute list, no `self` receiver, no
  parameter modes (`mut`, `shared`), and no default parameter values: these
  positions are absent from the grammar of an extern item and a program that
  writes them is ill-formed.
- A block's items are ordinary module-level declarations: each introduces one
  name, subject to every naming rule of [`names.md`](names.md) — a duplicate
  is `E0103`, a collision with another header binding is `E0104`, and a
  prelude-reserved spelling is `E0106`.
- An import block is **module-private unless marked `pub`**: `pub extern "C" { … }`
  exports every item of the block, and an item carries no individual `pub`.
  Visibility otherwise follows
  [`modules/visibility.md`](modules/visibility.md).
- `extern` blocks are declarations, not header forms: they appear in the body
  of a module, in source order with the other declarations.

### Export

```xulo
pub extern "C" fn add(a: i32, b: i32): i32 { a + b }   // ok
// pub extern "C" async fn tick() { }                  // error: an extern export has no `async` marker
// extern "C" fn helper(a: i32): i32 { a }             // error: an export form carries `pub`
```

- The form is `pub extern AbiTag fn name(params) [: Type] Block`. The
  declaration MUST carry `pub`: linkage with no export is not expressible.
- The exported **symbol is the declared name**. This version defines no way
  to export under a different symbol; in particular a symbol whose spelling
  is a Xulo keyword cannot be written ([Not in scope](#not-in-scope)).
- The body is an ordinary Xulo function body, and the function is an ordinary
  module-level `fn` for every other purpose: it may be called from Xulo code,
  passed as a value, and mentioned by `use`. Restricting an import to direct
  calls ([Calling](#calling)) applies only to imported symbols.
- The signature obeys the same type rules as an import item
  ([Types at the boundary](#types-at-the-boundary)).
- An extern export MUST NOT be `async`, generic, bounded by a `where` clause,
  attributed, or declared inside an `impl` block: those positions are absent
  from its grammar.

### Visibility and re-export

A `pub` import block exports its items, and `pub use` MAY re-export an
imported name like any other binding; the linkage is the symbol's property
and does not change with the name it is re-exported under. An imported name
that a module never uses is reported like any other unused binding.

## ABI tags

The ABI tag of a declaration is a string literal drawn from a **closed
registry**. The registry is a checking rule, not a grammatical one: the
grammar admits any string literal, and the checker validates it.

| Tag | Meaning |
|-----|---------|
| `"C"` | the C ABI of the target platform |
| `"system"` | the platform's default calling convention for system interfaces; on every target where the platform's system convention is its C convention, the two tags denote the same convention |

- Which platform ABI document `"C"` and `"system"` denote on a given target
  is implementation-defined but MUST be documented
  ([`memory-and-runtime.md`](memory-and-runtime.md)).
- A tag outside the registry is a compile-time error (`E0902`), whose note
  lists the registered set.
- Adding a tag to the registry in a later version is a minor change: no
  program that compiles today changes meaning.

## Types at the boundary

Only the following types may appear in an extern signature — as a parameter
type or as the return type. The list is closed.

| Xulo | Position | C counterpart |
|------|----------|---------------|
| `bool` | parameter, return | `_Bool`, per the platform ABI document |
| `i8` `i16` `i32` `i64` | parameter, return | `int8_t` … `int64_t` |
| `u8` `u16` `u32` `u64` | parameter, return | `uint8_t` … `uint64_t` |
| `f32` `f64` | parameter, return | `float`, `double` |
| `usize` `isize` | parameter, return | `size_t`, `ptrdiff_t` |
| `unit` | **return only** (or omitted) | `void` |
| `Ptr` | parameter, return | a pointer slot of the platform ABI |
| `Ptr?` | parameter, return | a nullable pointer slot |

- A type not in the table — including `string`, `List`/`Map`/`Set`, `struct`,
  `enum`, tuples, `View`, `Task`, `Result`, `unknown`, `number`, `int`,
  `float`, `null`, a union, an intersection, a function type, or a type
  parameter — in an extern signature is a compile-time error (`E0901`).
- `unit` as a **parameter** type is `E0901`: no C parameter maps to it.
- The only optional type in an extern signature is exactly `Ptr?`; `i32?`,
  `string?`, and `Ptr??` are `E0901`.
- The user-facing numeric layer (`number`, `int`, `float`) is excluded
  deliberately: a value crossing this boundary is written in the formal layer
  so that its width is on the page.

```xulo
extern "C" {
  fn sqrt(x: f64): f64
  fn strlen(p: Ptr): usize
  fn bad1(s: string): i32        // error[E0901]
  fn bad2(n: i32?): i32          // error[E0901]
  fn bad3(n: unit): unit         // error[E0901]: `unit` parameter
}
```

## Ptr

`Ptr` is the built-in type of an **opaque foreign handle**: a value that
names an address the language never lets a program inspect.

- `Ptr` takes **no type arguments**; a type application on it — `Ptr<u8>` —
  is ill-formed like any application to a non-generic type (`E0230`).
- A value of type `Ptr` is obtained only as the result of an extern call or
  by passing through Xulo code that received one. There is no literal, no
  cast, and no constructor for it.
- `Ptr` is **non-null**. A nullable handle — every C API that may return NULL
  — is `Ptr?`, and `Ptr?` is the ordinary optional type: it is exactly
  `Ptr | null`, and `??`, `?.`, `is`, `match`, and comparison against `null`
  behave on it as on every optional type. Nothing about it is special, and no
  `Ptr`-specific null-checking operation exists.
- The only operators on `Ptr` are `==` and `!=` (address identity).
  Ordering, arithmetic, and dereference are not defined on it.
- A `Ptr` is an ordinary value everywhere else: it MAY be bound with `let`,
  stored in a `struct` field, held in a `List`, returned from a Xulo function,
  and compared. The collector treats it as an opaque word; a `Ptr` designates
  foreign storage, never Xulo-managed storage, so no collection rule applies
  to what it names.
- **A `Ptr` never converts to or from an integer**, in this version or any
  later one.

```xulo
fn roundTrip(p: Ptr): Ptr? { p }          // ok: `Ptr` is an ordinary value

extern "C" {
  fn asInt(p: Ptr): i32                   // ok
  // fn asInt(p: Ptr<u8>): i32            // error[E0230]: `Ptr` takes no type arguments
  // fn boxed(p: Ptr): i32 { 0 }          // error: an import item has no body
}
```

### Handling `null`

Four forms, all inherited from `T?`:

```xulo
extern "C" {
  fn malloc(size: usize): Ptr? as cMalloc
  fn free(p: Ptr?) as cFree
}

fn store(size: usize) {
  let p = cMalloc(size)
  if p != null {
    // `p` is `Ptr` here
    cFree(p)                      // ok
  } else {
    print("out of memory")
  }
}

fn mustAllocate(size: usize): Ptr {
  let p = cMalloc(size) ?? panic("out of memory")   // ok: `p` is `Ptr`
  p
}
```

- **Narrowing is branch-local.** A comparison against `null` narrows the
  binding inside the branch that tests it; the specification has no
  flow-sensitive narrowing that survives an early exit. This is the ordinary
  rule of [`types/composite-types.md`](types/composite-types.md), and it
  surfaces here often enough to state:

```xulo
fn attempt(size: usize) {
  let p = cMalloc(size)
  if p == null { return }
  cFree(p)                        // error[E0201]: `p` is `Ptr?` here
}
```

  The accepted repairs rebind (`?? panic(…)` above) or move the use into the
  non-null branch. A `match` supplies both at once when each arm produces a
  value:

```xulo
fn touch(size: usize): string {
  match cMalloc(size) {
    Ptr p => { cFree(p); "freed" }    // ok: `p` is `Ptr` within this arm
    null => "allocation failed"
  }
}
```

## Calling

An imported symbol is callable in **direct call position** only: `cMalloc(64)`
is a call; binding it, storing it, or passing it as a value is a compile-time
error (`E0903`). The reason is structural — an Xulo value of function type
carries no ABI, so a call through it would use Xulo's own convention on a
foreign symbol. A function type that carries a foreign ABI is
[not in scope](#not-in-scope).

```xulo
fn apply(f: fn(i32): i32, x: i32): i32 { f(x) }

extern "C" { fn abs(x: i32): i32 }

fn main() {
  let n = abs(-1)                 // ok: direct call
  // let g = abs;                // error[E0903]
  // print(apply(abs, -1))       // error[E0903]
}
```

Everything else about a call is unchanged and is not restated here:
subexpressions and arguments evaluate left to right, named arguments and
omitted optional parameters are accepted as for any declared function — an
omitted `Ptr?` parameter binds `null` — and an argument is borrowed for the
call's duration as in any other call
([`memory-and-runtime.md`](memory-and-runtime.md)).

## The foreign boundary

A call to an imported symbol transfers control to code this specification
does not define. Accordingly:

- **The declaration side is fully checked.** The checker guarantees that an
  extern signature contains only ABI-safe types (`E0901`), that its ABI tag
  is registered (`E0902`), that imported names obey the naming rules, and
  that imports are called directly (`E0903`).
- **The callee side is outside this specification.** The behavior of foreign
  code during an extern call — what it reads, writes, allocates, or crashes —
  is not governed here, and no conformance requirement is imposed on it. What
  foreign code cannot do by construction is reach the Xulo heap or the
  bindings of a program: the arguments it receives are value copies and
  `Ptr` tokens, and it is never given a reference to Xulo-managed storage.
- This carve-out qualifies the claim in
  [`memory-and-runtime.md`](memory-and-runtime.md) ("this specification
  defines no undefined behavior") for exactly this boundary; that chapter
  states the qualification normatively.
- A foreign callee reaches Xulo code only by calling an exported symbol;
  such a reentrant call is an ordinary call and follows ordinary rules.

## Diagnostics

| Code | Trigger | Example | Message shape |
|------|---------|---------|---------------|
| `E0901` | a type that is not ABI-safe appears in an extern signature; `unit` as a parameter; an optional type other than `Ptr?` | `fn f(s: string): i32` in `extern "C" { … }` | `` `string` is not allowed across an `extern "C"` boundary `` — with `` note: ABI-safe types: `bool`, `i8`…`u64`, `f32`, `f64`, `usize`, `isize`, `unit`, `Ptr`, `Ptr?` `` |
| `E0902` | an ABI tag outside the registry | `extern "c++" { … }` | `` ABI `"c++"` is not registered `` — with `` note: registered ABIs: `"C"`, `"system"` `` |
| `E0903` | an imported symbol referenced outside direct call position | `let g = abs` | `` `abs` is an extern import and can only be called directly `` |

The family range is `E0901`–`E0999`, registered in
[`type-system/errors.md`](type-system/errors.md). Positions absent from the
grammar — a body on an import item, `async` on either extern form, a
parameter mode, a private export — are syntax failures and carry no code.

## Not in scope

The following constructs do **not** exist in this version. Their absence is
deliberate; each is deferred with the reason recorded alongside the change
that introduced this chapter ([`changes/`](changes/README.md)).

- **Linking and loading.** Which library a symbol is taken from, when it is
  opened, and how a manifest maps a package to a path are host and tooling
  concerns ([`README.md`](README.md), §1.1); this chapter defines the
  declaration, not the resolution.
- **Strings, buffers, and collections across the boundary.** No `string`,
  `List`, or `Map` parameter or return, and no borrowed buffer type.
- **Function pointers with a foreign ABI.** Foreign code cannot call back
  into Xulo through a value; it can only call an exported symbol.
- **Symbol names that differ from the declared name on export**, including
  symbols whose spelling is a Xulo keyword.
- **Struct layout, and any type whose layout the language does not
  otherwise specify.**
- **ABI tags other than `"C"` and `"system"`** — including `"c++"`, which is
  not expected to be registered, and `"wasm"`, which awaits a compilation
  target.
- **Any conversion between `Ptr` and an integer, and any operation that
  dereferences a `Ptr`.**

## Example

```xulo
extern "C" {
  fn abs(x: i32): i32
  fn malloc(size: usize): Ptr? as cMalloc
  fn free(p: Ptr?) as cFree
}

pub extern "C" fn sum(xs: i32, n: i32): i32 {
  let mut total: i32 = 0
  let mut i: i32 = 0
  while i < n {
    total = total + abs(xs + i)   // ok: direct calls on both symbols
    i = i + 1
  }
  total
}

fn withBuffer(size: usize, body: fn(Ptr): i32): i32 {
  let p = cMalloc(size) ?? panic("out of memory")
  let result = body(p)            // Xulo closure over a `Ptr`: ordinary call
  cFree(p)
  result
}
// ok
```

