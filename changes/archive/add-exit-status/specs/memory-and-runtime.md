# Memory and Runtime — Delta: Exit Status

> **Delta** — adds a new `## Exit status` section in `memory-and-runtime.md`
> between `## Runtime failures` and `## Determinism`, and replaces the third
> bullet of the “Implementation-defined” list; everything else is unchanged.

## Runtime failures

(unchanged — the section ends with its closing paragraph, followed by the
new section below)

## Exit status

The language has no I/O and no `exit` function; how a program ends is a
property of the program and the host environment
([`README.md`](README.md), § 1.1).

- A program that terminates normally — `main` returns `unit`, the task
  returned by an `async` `main` settles, or the presentation target of a
  `View` program closes — exits with status **0**.
- A runtime failure that ends the program — a panic, and with it any
  condition of the closed tables above — exits with a **nonzero** status.
  The exact nonzero values, and how the failure is presented to the
  environment, are implementation-defined
  ([Implementation-defined](#implementation-defined)).

## Determinism

(unchanged)

### Implementation-defined

Choices an implementation MAY make but MUST document — four, all at or
beyond the boundary between the program and its environment:

- the recursion and live-task limits behind *Stack or task exhaustion* and
  *Task exhaustion*, and whether anything outside the language can raise
  them;
- how exhaustion of resources the language does not model — available memory
  above all — is detected and reported;
- how a runtime failure is presented to that environment — message text,
  stream, and the exact nonzero exit status — since the language itself has
  no I/O ([`README.md`](README.md), § 1.1);
- which ABI document each registered tag (`"C"`, `"system"`) denotes on the
  target platform — and therefore the pointer width `W` that `usize` and
  `isize` have, and the layout every ABI-safe signature is laid out in
  ([`ffi.md`](ffi.md), [`types/primitive-types.md`](types/primitive-types.md)).
