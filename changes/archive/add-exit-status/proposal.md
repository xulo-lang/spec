# Change Exit Status

**Status:** accepted · **Date:** 2026-10-23 · **Affects:**
`memory-and-runtime.md`, `modules/source-files.md`, `README.md`

## Summary

The specification says how a program starts (`main`, the signatures it may
have) and how it fails (the closed runtime-failure tables), but nowhere
says what the host observes when it ends. Two conforming implementations
can therefore hand the process a different exit code for the same
program — the success signal every script and test harness reads. This
change pins exactly the minimum the language owes the host:

> A program that terminates normally exits with status **0**; one that
> ends in a runtime failure exits with a **nonzero** status. The exact
> nonzero values remain implementation-defined and documented.

It deliberately adds no `exit` function, no argv or stream API — the
language has no I/O, and those belong to the standard library and FFI.
The deltas in [`specs/`](specs/) are the text the chapters gain once the
change is accepted.

## Motivation

### The success signal is part of the observable behavior a bootstrap depends on

A self-hosted test runner compiled from Xulo must be able to report pass
or fail to whatever launched it. If the exit code of a normal return is
unspecified, two implementations disagree about the single bit the
harness reads, and a bootstrap cannot be validated across them. Freezing
v1 means freezing what a conforming host may observe; "the program ended
well" is the first-order observation.

### The spec already owns every adjacent fact

`memory-and-runtime.md` owns what a runtime failure is (a panic, no
handler) and already lists "exit status" among the things an
implementation MAY choose how to present — while never stating what that
status is on the normal path. `modules/source-files.md` owns `main` and
its signatures. The three passages together leave a gap exactly at the
boundary; closing it is a sentence per chapter, not a new subsystem.

### The cheap fix is Go's shape, and the spec already has half of it

Go pins exit 0 / nonzero and leaves the failure codes to the
implementation. The spec's implementation-defined list already
carries the presentation question ("message text, stream, exit
status"); narrowing "exit status" to "the exact nonzero exit status"
is the whole edit on that side.

## Scope

**In scope.** The exit-status rule in `memory-and-runtime.md` (new
subsection, plus the narrowed implementation-defined bullet); the
cross-reference bullet in the `main` section of
`modules/source-files.md`; the informative self-hosting note in
`README.md` § 1.1.

**Out of scope.** An `exit(code)` intrinsic (standard library or FFI
concern); argv, standard streams, and file access in the language; signal
handling; distinguishing *which* runtime failure occurred in the status;
any statement about the `View` program's exit code after the target is
already closed.

## Compatibility

No conforming program can regress: nothing in the current chapters
assigns an exit status, so no program has a defined meaning to lose. An
implementation that already exits 0 on normal return — the only sensible
choice — gains a guarantee; one that did not was non-conforming under the
closed-list claim of `memory-and-runtime.md` anyway.

## Non-goals

- No `exit` intrinsic in the language; `extern "C" { fn exit(code: i32):
  unit }` remains available through the FFI of [`ffi.md`](ffi.md).
- No table of named failure codes; nonzero is the whole rule.
- No exit status for foreign exports — a C-ABI export returning a value
  is its own contract with the caller.

## Readiness

- [x] Depends on nothing unaccepted; every delta is written against the
      current, folded chapters.
- [x] All three affected passages are covered by [`specs/`](specs/) and
      [`tasks.md`](tasks.md), including the implementation-defined bullet
      that must be narrowed in step.
