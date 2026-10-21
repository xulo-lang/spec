# Memory and Runtime — Delta: The Foreign-Boundary Carve-Out

> **Delta** — edits the **Undefined and implementation-defined behavior**
> section of `memory-and-runtime.md`.

## The zero-UB claim

The opening claim gains one scope sentence, after “At no point is an
implementation permitted to continue in a way of its own choosing, and no
operation has a result, order, or effect that the specification leaves open
to it.”:

> One boundary is outside this promise, and is the only one: during an extern
> call declared in [`ffi.md`](ffi.md), what the foreign code itself does —
> memory it touches, values it produces, effects it has on the host — is
> neither specified nor checked here. The Xulo side of that call remains
> inside the promise: every argument expression, every conversion to an
> ABI-safe type, and the result conversion are fully specified, and an
> implementation MUST document which ABI document its registered tags denote
> (below). Nothing else in the language crosses this boundary.

The “two categories” sentence that follows stays exactly as it is: the
carve-out is not a third category — it is a statement that one class of
*foreign* behavior was never a property of Xulo programs at all. The two
lists remain closed for every construct of the language.

## Implementation-defined

The count becomes **four, all at or beyond the boundary between the program
and its environment**, and a fourth bullet joins the list:

- which ABI document each registered tag (`"C"`, `"system"`) denotes on the
  target platform — and therefore the pointer width `W` that `usize` and
  `isize` have, and the layout every ABI-safe signature is laid out in
  ([`ffi.md`](ffi.md), [`types/primitive-types.md`](types/primitive-types.md)).

## Unspecified

Unspecified — behavior fixed neither by this specification nor by any
requirement to — is untouched: nothing about a Xulo program's own execution
becomes unspecified by this change. The foreign carve-out is not listed here,
because it is not a property of the Xulo program being run.
