# Introduction — Delta: The Foreign-Interface Non-Goal

> **Delta** — replaces the “No foreign-function interface contract” bullet of
> the **Non-Goals** list in `introduction.md`.

- **No C++ or foreign-runtime ABIs.** The language declares foreign functions
  against exactly two ABI tags, `"C"` and `"system"`, and nothing else
  ([`ffi.md`](ffi.md)): C++ interop — which has no single ABI, Itanium and
  MSVC name mangling being incompatible — and calling into managed runtimes
  remain library and host concerns. The boundary carries value copies and
  opaque handles only: no string, buffer, or callback crosses it, and no
  foreign ABI annotation exists beyond the registry.

Editing note — not chapter text: the bullet this replaces — “No
foreign-function interface contract” — is removed whole; the new bullet above
takes its place in the list. The earlier bullet “Not a replacement for a
systems language” is untouched and now reads as its intended counterpart: the
fixed-bit types exist for ABI and FFI precision, and [`ffi.md`](ffi.md)
states the boundary line from the other side — a `Ptr` is never
dereferenceable and never converts to an integer.
