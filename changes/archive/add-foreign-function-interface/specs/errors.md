# Type Errors and Diagnostics — Delta: The Foreign-Boundary Family

> **Delta** — adds a row to the **Error code ranges** table and a
> **Foreign boundary errors** subsection to `type-system/errors.md`, between
> **Trait and generic errors** and **Warnings**.

The ranges table gains:

| Range | Family | Specified in |
|-------|--------|--------------|
| `E0901`–`E0999` | foreign boundary: extern signatures, ABI tags, extern calls | [`../ffi.md`](../ffi.md) |

The new subsection, inserted between **Trait and generic errors** and
**Warnings**:

### Foreign boundary errors

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E0901` | a type that is not ABI-safe appears in an extern signature; `unit` as a parameter; an optional type other than `Ptr?` | `fn f(s: string): i32` inside `extern "C" { … }` | `` `string` is not allowed across an `extern "C"` boundary `` | [ffi.md](../ffi.md) |
| `E0902` | an ABI tag outside the registry | `extern "c++" { … }` | `` ABI `"c++"` is not registered `` | [ffi.md](../ffi.md) |
| `E0903` | an imported symbol referenced outside direct call position | `let g = abs` | `` `abs` is an extern import and can only be called directly `` | [ffi.md](../ffi.md) |

`E0901` carries the note `` = note: ABI-safe types: `bool`, `i8`…`u64`,
`f32`, `f64`, `usize`, `isize`, `unit`, `Ptr`, `Ptr?` ``, and `E0902`
carries `` = note: registered ABIs: `"C"`, `"system"` `` — both notes follow
the `E0111` pattern. Positions that do not exist in an extern form's grammar
— a body on an import item, `async` on either form, a parameter mode, a
private export — are syntax failures raised before the checker runs and carry
no code (see **Error categories** above).
