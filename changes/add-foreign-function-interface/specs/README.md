# Xulo Language Specification — Delta: Document Map and Terminology for `ffi.md`

> **Delta** — edits § 1 (purpose list), § 4 (Document Map), and § 5
> (Terminology) of `README.md`.

In § 1, the sixth bullet becomes:

> - the memory, concurrency, error-handling, and foreign-boundary models,

In § 4, one row joins the document map, after
[`memory-and-runtime.md`](memory-and-runtime.md):

| Document | Contents |
|----------|----------|
| [`ffi.md`](ffi.md) | The foreign boundary: `extern` imports and exports, the ABI registry, ABI-safe signature types, `Ptr`, and the boundary guarantee |

In § 5, three terminology entries join the table, each in
case-insensitive alphabetical position:

| Entry | Text |
|-------|------|
| **ABI-safe** | Of a type permitted in an `extern` signature: on the closed list of [`ffi.md`](ffi.md) — `bool`, the fixed-bit types, `usize`/`isize`, `unit`, `Ptr`, and `Ptr?` — so that it has a defined representation under a registered ABI. |
| **extern** | The declaration forms that cross the foreign boundary: `extern "ABI" { fn …; }` imports symbols, `pub extern "ABI" fn …` exports one. Only value copies and opaque handles cross; see [`ffi.md`](ffi.md). |
| **Ptr** | An opaque, non-null handle to memory the language does not own: produced only by an extern call or passed back through one, comparable only for identity, and never dereferenceable. See [`ffi.md`](ffi.md). |

`Ptr` sits between **pattern** and **pub**; `extern` between **evaluation
order** and **immutable binding**; `ABI-safe` before **binding**.

The **declaration** entry — “A top-level or member-level construct that
introduces a name: `fn`, `struct`, `enum`, `trait`, `impl`, `type`, `let`,
`const`.” — is unchanged: an `extern` block is not a declaration of a name of
its own, it is a container of import items
([`grammar.md`](grammar.md)).
