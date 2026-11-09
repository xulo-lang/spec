# Design: Test Options

**Status:** accepted · **Date:** 2026-11-06
**Change ID:** `add-test-options`

## Goals

1. Let one test declaration state whether it expects normal completion, any
   failure, or a panic specifically.
2. Let a test declaration be skipped, with an optional human-readable reason.
3. Let a test declaration set a per-test deadline in milliseconds.
4. Make `should_fail` stable as the general entry point even if a future test
   form adds a failure mode that is not a panic.
5. Keep the change additive and small: no new error code, no change to
   `panic`, runtime failures, or the basic shape of a test run.

## Non-goals

- `category`, `focus`, `retry`, and `threads`. They need filtering, selection,
  failure, and scheduling policies that do not exist yet; this change leaves
  the option group extensible for them but rejects them today with `E0110`.
- A returned-error test channel. Test bodies remain `fn(): unit`; `Result`
  values are ordinary data.
- Catching panics. `panic` remains unhandleable; the test runner observes the
  outcome of an isolated run, not a language-level handler.
- Fixing runtime-failure message text. That remains implementation-defined and
  documented.
- General attribute values. The accepted `add-macro-system` change already
  supplies the widened value grammar; this change adds only the test-option
  grammar.

## Grammar

### Option group

The test marker currently has a fixed literal head and no group. The change
makes the group optional and gives the test form its own option productions
rather than reusing the unrestricted attribute argument list:

```text
TestDecl = "#" "[" "test" [ "(" TestOptions ")" ] "]" [ "async" ] "fn"
           Identifier "(" ")" Block ;
TestOptions = TestOption { "," TestOption } [ "," ] ;
TestOption = TestExpectation | "skip" [ "=" StringLiteral ]
           | "timeout" "=" IntegerLiteral ;
TestExpectation = "should_fail" | "should_panic" [ "=" StringLiteral ] ;
```

`TestOptions` is a non-empty comma-separated list; a trailing comma is
allowed. The list is parsed without regard to order. These validity rules are
part of recognizing a `TestDecl` and a violated rule is `E0110`:

- at most one `TestExpectation`;
- at most one `timeout`;
- at most one `skip`, and `skip` MUST be the only option in its group;
- every option must come from `TestOption`;
- `timeout`'s value must be an integer literal whose value is greater than
  zero.

A group with neither an expectation nor `skip` is a normal test; a group
with only `timeout` is a normal test with a deadline. A group with
`should_panic = "text"` carries the expected substring. A group with `timeout`
and one expectation is valid in either order. An empty group is not a `TestOptions`
list and does not match `TestDecl`.

### Attribute-argument dependency

The accepted `add-macro-system` change supplies the widened attribute
grammar, `AttributeArg = [ Identifier "=" ] AttributeValue` with identifier,
string, integer, float, and boolean values. This change does not alter it. The
widened grammar lets the fallback path parse `#[test(skip = "…")]`,
`#[test(timeout = 100)]`, and the other option groups when the complete test
form does not match, so the user gets `E0109` or `E0110` instead of a parse
error.

### Why a dedicated option grammar

**Chosen: dedicated productions inside `TestDecl`.** The test marker is
already a special declaration form, not a generic attribute on an ordinary
`fn`: its host shape is fixed and its name binds no value. Keeping its
argument grammar fixed makes unknown options and wrong value types diagnosable
as `E0110` rather than making the checker police a general argument list. The
generic `AttributeArg` widening is only for the fallback and the accepted
macro system.

**Rejected: `TestDecl = ... [ "(" AttributeArgs ")" ] ...`.** It would parse
`#[test(should_fail, should_panic, "x")]` as a test declaration and then
require a long list of semantic checks. The dedicated productions state the
accepted shapes once, in the grammar, and keep the list extensible.

**Rejected: a standalone `#[should_panic]`/`#[skip]` attribute.** Several
attributes would make ordering, duplication, and combination rules harder,
and would allow `#[test]` without an expectation beside a stray marker. One
marker with one option group keeps the declaration self-contained.

**Rejected: one option only.** A single `TestExpectation | skip | timeout`
alternative is smaller, but it makes `timeout` unusable with an expectation
and forces a grammar change as soon as a second orthogonal option appears.
The list shape is the extension point for `category`, `focus`, and the other
deferred options.

## Semantics

### Outcomes

A test run ends in exactly one of four outcomes:

- **completed** — the body runs to completion; for `async`, the started task
  settles as completed.
- **panicked** — a `panic` or a runtime failure stops the body. The
  specification already treats a runtime failure as a panic raised by the
  implementation.
- **cancelled** — an `async` test's task settles as cancelled while the test
  is running; if a configured deadline expires first, the outcome is
  `timedOut` instead.
- **timedOut** — a configured `timeout` deadline expires before the body
  settles. This is a failure but not a panic.

A skipped test does not run and has no outcome; the tool reports it as
skipped.

The modes select outcomes:

| Mode | Passes on | Fails on |
|------|-----------|----------|
| normal (`#[test]`) | completed | panicked, cancelled, timedOut |
| `should_fail` | panicked, cancelled, timedOut | completed |
| `should_panic` | panicked | completed, cancelled, timedOut |
| `should_panic = "text"` | panicked with a matching message | completed, cancelled, timedOut, non-matching panic |

This makes `should_panic` a strict subset of `should_fail` today, while
keeping the two rules independently meaningful. `should_fail` is defined by
the complement of the normal pass condition rather than by an enumeration of
panic causes, so a later failure mode — a returned `Err`, for example — joins
it without a new attribute.

A timeout is a failure, not a panic. Consequently `should_fail` passes when a
test times out, and `should_panic` fails. A test that must fail for a
particular reason should use `should_panic` with a message rather than
`should_fail` with a short timeout, since the timeout itself satisfies
`should_fail`.

### Skip

`skip` means the test is not executed. Its body is still checked normally,
its name is still not referenceable (`E0107`), and it does not suppress any
other compile-time error. A skipped test is reported as skipped, with the
reason when one is written; it is neither a pass nor a failure.

`skip` is the only option in its group. Combining it with an expectation or
`timeout` is `E0110`, because those options describe a run that will not
happen. A future metadata option such as `category` may reasonably combine
with `skip`; this change keeps the option list shape so that later change can
relax the rule without redefining skip.

### Timeout

`timeout = N` sets a per-test wall-clock deadline of `N` milliseconds. `N` is an integer
literal greater than zero. The deadline starts when the test body begins
executing; it includes time spent in `await` and in ordinary execution. It does
not include discovery, queueing, or tool overhead before the body starts.

If the body settles before the deadline, the run has its normal outcome. If
the deadline expires first, the outcome is `timedOut`; the tool stops waiting
and reports the timeout. For an `async` test, the tool cancels the task. How a
tool reclaims a synchronous body that ignores the deadline is outside the
language, but it MUST stop waiting and MUST NOT later report a second outcome
for that run.

### Message matching

`should_panic = "text"` matches the failure message:

- For `panic(message)`, the message is the evaluated `message` string.
- For a runtime failure, the message is the implementation's description of
  that failure. Its wording is implementation-defined and documented with the
  other implementation-defined presentation choices
  ([`memory-and-runtime.md`](../../memory-and-runtime.md#implementation-defined)).
- The expected text is the cooked value of the string literal. The failure
  message matches when it contains that text as a case-sensitive substring.
  It is not a glob, regular expression, or exact match. An empty string is a
  substring of every message and therefore matches every panic.

The substring choice is deliberately permissive. Exact equality would make
tests brittle against contextual prefixes and a runtime failure's diagnostic
wrapper; a pattern language would need its own grammar, escaping, and
diagnostics. A test that needs a portable message should panic explicitly
with that text rather than depend on a runtime failure's implementation-defined
wording.

### Interaction with `async`

Cancellation is a failure but not a panic. It passes `should_fail` and fails
`should_panic`, including `should_panic = "…"`. This is the clearest current
behavioural difference between the two modes. A panic inside an `async` body
is still a panic; it passes either mode, subject to message matching.

A timeout of an `async` test cancels its task, but the run outcome is
`timedOut`, not `cancelled`. If the task settles as cancelled before the
deadline, the outcome is `cancelled`; if the deadline expires first, it is
`timedOut`.

### Invalid groups

`E0110` remains the code for a test marker whose option shape is wrong. The
message names the accepted forms:

```text
attribute `test` takes `should_fail`, `should_panic`,
`should_panic = "…"`, `skip`, `skip = "…"`, or `timeout = N`
```

The following are `E0110` when the marker is on a `fn` that otherwise fails
the test form, or when the complete test form fails but the group still parses
as an ordinary attribute:

- `#[test()]`
- `#[test(should_fail = "x")]`
- `#[test(should_panic, should_fail)]`
- `#[test(should_panic, should_panic)]`
- `#[test(should_panic = "x", other)]`
- `#[test(other)]`
- `#[test(skip, should_fail)]`
- `#[test(skip, timeout = 100)]`
- `#[test(skip, skip = "x")]`
- `#[test(timeout = 0)]`
- `#[test(timeout = "100")]`
- `#[test(category = "slow")]`, `#[test(focus)]`, `#[test(retry = 3)]`,
  `#[test(threads = 1)]` — deferred option names

A positional string, numeral, float, or boolean — `#[test("x")]`,
`#[test(1)]`, `#[test(1.5)]`, `#[test(true)]` — parses as an ordinary
attribute under the accepted widened `AttributeArg` and is `E0110`. A
negative number is not grammatical and remains a syntax error. Written on a
`struct`, `enum`, `trait`, or `impl`, the marker keeps its existing `E0109`
host error regardless of the group.

## AST

The options are metadata of the test declaration, not generic attributes:

```typescript
type TestMode = 'normal' | 'shouldFail' | 'shouldPanic';

interface TestDeclaration extends BaseNode {
  type: 'TestDeclaration';
  name: IdentifierNode;
  isAsync: boolean;
  mode: TestMode;
  expectedMessage: StringLiteral | null;
  isSkipped: boolean;
  skipReason: StringLiteral | null;
  timeoutMs: NumberLiteral | null;
  body: BlockNode;
}
```

- `mode` is `normal` when the marker has no expectation, `shouldFail` for
  `should_fail`, and `shouldPanic` for both `should_panic` forms.
- `expectedMessage` is non-null only for `should_panic = "…"`; it carries the
  literal's span and cooked text, so a tool can report the expectation
  without re-reading source.
- `isSkipped` is true only when `skip` is written. Because `skip` is alone,
  a skipped declaration has `mode: 'normal'`, `expectedMessage: null`, and
  `timeoutMs: null`.
- `skipReason` is non-null only for `skip = "…"`; it is the cooked string.
- `timeoutMs` is non-null only for `timeout = N`. It is a `NumberLiteral` of
  kind `integer`; the checker rejects zero or a non-integer value.
- No `attributes` list is added. The marker is still a declaration form, and
  its options are not ordinary attributes that name resolution or macro
  expansion can see.

**Rejected: storing the group as an `AttributeNode`.** It would expose
`should_fail`/`should_panic`/`skip`/`timeout` as attribute arguments, make
`#[test]` and its metadata two different mechanisms, and force tools to know
that this one attribute's arguments are special. The dedicated fields make
the test form's contract explicit.

**Rejected: one `expectation` child node.** The group has no independent
semantic identity; it selects one of three modes and may carry a literal.
Fields are smaller than a node whose only child is a string literal.

### Attribute argument nodes

The accepted `add-macro-system` change already defines the fallback nodes;
they are repeated here only to show the shape this change relies on:

```typescript
interface AttributeNode extends BaseNode {
  type: 'Attribute';
  name: AttributeNameNode;
  args: AttributeArgNode[] | null;
}

interface AttributeArgNode extends BaseNode {
  type: 'AttributeArg';
  name: IdentifierNode | null;
  value: AttributeValueNode;
}

type AttributeValueNode = IdentifierNode | StringLiteral | NumberLiteral;
```

`args` is null for a bare attribute and an array when a parenthesized group is
written; the array is empty for `#[test()]`. An `AttributeArgNode` with a
non-null `name` is the `name = value` form; a bare identifier argument has
`name: null` and an `IdentifierNode` value. `NumberLiteral` values are
integers in the test options. The accepted `add-macro-system` change broadens
`AttributeValueNode` with positional values, floats, and booleans; this change
keeps the node shape unchanged.

**Rejected: `skipReason` alone without `isSkipped`.** `skip` without a reason
and no skip at all would both have a null reason. The boolean makes the
distinction explicit.

## Diagnostics

No new code is introduced.

- `E0110` covers every malformed option group; the existing
  “registered attribute whose arguments do not fit the registered shape”
  trigger fits unchanged.
- `E0109` still covers the marker written on a host that has no test form.
- `E0107` still covers references to the test name.
- A negative number remains a syntax error because this change does not add
  unary-minus value forms.

The test form is recognized before attribute lookup. A complete
`#[test(should_panic = "…", timeout = 100)] async fn name() { … }` is a
`TestDecl`; a marker that fails the declaration shape is an ordinary
`Attribute` for diagnostic purposes, which is why the named string and named
integer arguments must exist in the generic argument grammar too.

## Compatibility and rebase

- Existing `#[test]` declarations are unchanged in meaning and AST except for
  the new fields, which are `normal`, `null`, `false`, `null`, and `null`.
- Existing invalid groups keep their outcome. `#[test(should_fail)]` was
  previously an ordinary attribute and `E0110`; it is now a complete test
  form. `#[test(should_panic = "…")]` and `#[test(timeout = 100)]` were
  previously syntax errors or `E0110`; they are now complete test forms when
  the declaration shape matches, and `E0110` when it does not.
- The accepted `add-macro-system` change supplies the wider `AttributeArg` and
  the same `AttributeArgNode` shape, including the named string and named
  integer alternatives this change needs. This change leaves those
  productions and nodes unchanged. Because the accepted grammar also admits
  positional values, `#[test("x")]` parses as an ordinary attribute; on a
  test-shaped `fn` that is the existing `E0110` fallback, not a new accepted
  test form.

## Alternatives considered

### `should_fail` as an alias of `should_panic`

Simple, but it makes the general mode depend on the failure cause. A future
returned-error test, or a cancelled `async` test, would either pass the wrong
mode or force a breaking change. Rejected.

### Exact message equality

Stricter, but brittle: runtime failures add context, and an implementation may
wrap a message. Substring matching is the conventional middle ground and
matches the requested “contains” semantics. Rejected exact equality.

### Boolean fields instead of one mode

`should_fail: bool` plus `should_panic: bool` would admit contradictory
states. One `TestMode` union makes the states unrepresentable. Rejected.

### A separate `#[should_fail]` attribute

It would allow a test marker and an expectation marker to drift apart, and
would require ordering/combination rules. One marker with one optional group
keeps the declaration self-contained. Rejected.

### Reusing `AttributeArgs` directly

It would parse invalid combinations and require the checker to re-state the
test shape. Dedicated option productions make the accepted language visible
in the grammar. Rejected.

### Timeout as a mode

`timeout` is orthogonal to what outcome is expected: a normal test, a
`should_fail` test, and a `should_panic` test can all have a deadline. Making
it a fourth mode would exclude useful combinations. Rejected.

### Timeout as a panic

Treating a timeout as a panic would let `should_panic` pass on a deadlock,
which is the opposite of what a timeout is for. It is a distinct failure
outcome. Rejected.

### Timeout as a string

A string would need parsing and range checks in the checker and would invite
units such as `"100ms"`. The option is milliseconds by definition, so an
integer literal is smaller. Rejected.

### Skip as an expectation

`skip` does not describe an outcome; the test never runs. Modelling it as a
fourth `TestMode` would make “skipped and should_fail” representable. A
separate boolean avoids that. Rejected.
