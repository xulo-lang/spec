# Functions — Delta: Explicit Type Arguments

> **Delta** — replaces the **Call-site inference** bullet of “Generic functions” in `functions.md`.

- **Call-site type arguments.** A call MAY write the type arguments of a
  generic function explicitly, in a list between the callee and the argument
  list — `first<Int>(…)` — or it MAY write none; there is no partial form.
  When the list is written it replaces inference for that call: its length
  MUST equal the declared type-parameter count (`E0808`), every written
  argument MUST satisfy its parameter's bounds (`E0801`), no expected type may
  override a written argument, and a list before a callee that declares no
  type parameters is `E0807`. When the list is omitted, type arguments are
  inferred from the arguments and the expected type, and every inferred
  argument MUST satisfy the parameter's bounds.
