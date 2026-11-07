# Add Test Options

**Status:** draft · **Date:** 2026-11-06
**Change ID:** `add-test-options`
**Affects:** `grammar.md`, `ast.md`, `testing.md`, `types/traits.md`,
`changes/README.md`

## Summary

The test marker gains one optional comma-separated option group. This change
covers the first four practical extensions: `should_fail`, `should_panic`,
`skip`, and `timeout`.

| Marker | Meaning |
|--------|---------|
| `#[test]` | the body completes normally |
| `#[test(should_fail)]` | the body fails for any reason |
| `#[test(should_panic)]` | the body stops by a `panic` or a runtime failure |
| `#[test(should_panic = "text")]` | it stops by a `panic` or runtime failure whose message contains `text` |
| `#[test(skip)]` | the test is not executed |
| `#[test(skip = "reason")]` | the test is not executed; `reason` is reported |
| `#[test(timeout = 100)]` | the run must finish within 100 milliseconds |

`should_fail` and `should_panic` are deliberately not synonyms.
`should_fail` is the general inverse of a passing test: it accepts any
failure the test form can report, including a cancelled `async` task and a
timeout. `should_panic` is stricter: it requires the failure to be a panic —
Xulo already treats a runtime failure as a panic raised by the
implementation — and rejects a cancelled task, a timeout, or any future
non-panic failure. A message argument narrows `should_panic` further to a
case-sensitive substring of the failure message.

`skip` is a run directive, not an expectation: a skipped test is not executed
and has no pass/fail outcome, but its body is still checked. `timeout` is a
wall-clock run limit: an expired deadline is a failure, but it is not a panic,
so it passes `should_fail` and fails `should_panic`.

The options are independent where that is useful: one expectation may be
combined with `timeout`, in either order. `skip` is the only option in its
group; combining it with an expectation or `timeout` is `E0110`.

```xulo
#[test]
fn addition_works() {
  assert(add(1, 1) == 2, "one plus one should be two")
}

#[test(should_fail)]
fn addition_fails_as_expected() {
  assert(add(1, 1) == 3, "one plus one is three")
}

#[test(should_panic)]
fn division_by_zero_panics() {
  divide(1, 0)
}

#[test(should_panic = "division by zero")]
fn division_by_zero_message() {
  divide(1, 0)
}

#[test(skip = "not implemented yet")]
fn future_feature() {
  assert(new_api() == true)
}

#[test(timeout = 100)]
fn must_finish_quickly() {
  heavy_computation()
}
```

## Motivation

A test suite needs to exercise failure paths, not only successful ones. The
current `#[test]` form has one criterion — the body must complete — so a test
of a rejected input, a broken invariant, or a deliberate panic cannot express
what it expects. The runner sees a failure and reports it, even when the
failure is the point of the test.

Two expectations cover the useful cases without pretending they are the same:

- **`should_fail`** is about the outcome, not its cause. A failed assertion,
  a deliberate `panic`, a division by zero, a cancelled `async` task, or a
  timeout all count. It is the general entry point a test suite reaches for
  first. The name also stays useful if the test form later gains an
  error-returning failure: a returned `Err` would be another way not to
  complete.
- **`should_panic`** is about the cause. It says the body must stop by a
  panic or by a runtime failure, which this specification already defines as
  the implementation raising a panic. A cancelled task is not a panic, a
  timeout is not a panic, and a future returned `Err` would not be one either.

Two run controls are also needed before the runner grows filtering and
scheduling features:

- **`skip`** records a known-not-running test in the source instead of
  deleting it or commenting it out. The reason is for the person reading the
  report, not for the language.
- **`timeout`** gives a per-test deadline. A hung or accidental loop should
  not be able to stall the whole suite, and the intent belongs beside the
  test rather than only in a runner flag.

Today Xulo test bodies return `unit`, so there is no returned-error failure
channel; `should_fail` and `should_panic` coincide for most current tests. The
distinction is still worth fixing now: it documents intent, keeps
`should_fail` stable under a future error-returning test form, and gives
message matching a clear home in `should_panic` rather than overloading the
general mode.

The deferred options are useful too, but they depend on a more mature runner:
`category` needs filtering, `focus` needs a run-selection policy, `retry`
needs a failure policy, and `threads` needs a scheduler. This change does not
freeze their syntax. For now those names are ordinary unknown options and
produce `E0110`; a later change can add them without changing the option
group's shape.

## Scope

**In scope.**

- The optional option group on `TestDecl`: expectations, `skip`, and
  `timeout`.
- The named string and named integer attribute arguments those options need.
- The attribute-argument AST nodes the widened grammar needs.
- The test AST fields carrying mode, expected message, skip reason, and
  timeout.
- The declaration, checking-independent pass/fail criterion, message matching,
  skip behaviour, timeout outcome, and the interaction with `async`
  cancellation.
- Reusing `E0110` for a malformed option group; no new diagnostic.

**Out of scope.**

- `category`, `focus`, `retry`, and `threads`. They remain unsupported and
  currently produce `E0110`.
- A new test return type or returned-error failure channel. A test body still
  returns `unit`; a helper's `Result` is data and is not automatically a test
  failure.
- Changing when `panic` or a runtime failure stops execution, or adding a way
  to catch one.
- Fixing the wording of runtime-failure messages. Their text stays
  implementation-defined and documented
  ([`memory-and-runtime.md`](../../memory-and-runtime.md#implementation-defined)).
- The scheduler and reporting around a run. Timeout defines the outcome and
  the deadline; reclaiming a synchronous body that ignores the deadline is
  outside the language.
- Test discovery, filtering, isolation, and running order; those remain tool
  concerns.

## Delivery priority

1. `#[test]` and its existing run rules.
2. `should_fail`, `should_panic`, and `should_panic = "text"`.
3. `skip` and `skip = "reason"`.
4. `timeout = N`.
5. Later: `category`, `focus`, `retry`, and `threads`, once the runner has
   filtering, selection, failure, and scheduling policies for them.

## Compatibility

The change is additive. Every existing `#[test]` declaration keeps its
meaning. The only existing spellings whose classification changes are test
markers with arguments: those forms were previously rejected with `E0110`
(except forms the attribute grammar could not parse, which were syntax
errors). The recognized option forms now make a complete test declaration;
the remaining malformed groups keep the same `E0110` or syntax-error outcome.

`add-macro-system` also widens `AttributeArg` and introduces the same
attribute-argument AST shape. This change needs named strings for
`should_panic` and `skip`, and named integers for `timeout`. The grammar and
AST deltas must be merged when either change is accepted, not treated as
alternatives.

## Open Questions

None blocking the draft. The exact text of a runtime failure's message stays
implementation-defined; tests that need a portable message assertion should
panic explicitly. How a tool stops a synchronous body at its deadline is
implementation-defined, but the reported outcome is still `timedOut`.
