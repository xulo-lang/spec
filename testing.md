# Tests

A program's tests are declarations in its source: the `#[test]` form marks a
function to be checked with the rest of the file and to be executed only by a
test tool. This chapter specifies the declaration's form, when its body is
checked, what the `async` marker adds, and what it means for a test to pass or
fail. Which tests a tool discovers, the order it runs them, what each run is
isolated from, and how a result is reported are properties of the tool, not of
this specification ([README §1.1](README.md)).

## Declaration

The form is `TestDecl` in the grammar ([grammar.md](grammar.md)):

```text
TestDecl = "#" "[" "test" [ "(" TestOptions ")" ] "]" [ "async" ] "fn"
           Identifier "(" ")" Block ;
TestOptions = TestOption { "," TestOption } [ "," ] ;
TestOption = TestExpectation | "skip" [ "=" StringLiteral ]
           | "timeout" "=" IntegerLiteral ;
TestExpectation = "should_fail" | "should_panic" [ "=" StringLiteral ] ;
```

- A `#[test]` function appears at file scope, among a module's top-level
  declarations ([modules/source-files.md](modules/source-files.md)); it is
  not a statement and appears nowhere else.
- It carries no `pub`, no parameters, no return type, no generic parameters,
  and no `where` clause: the body is an implicit `fn(): unit`, and the
  declaration is never part of an interface
  ([modules/visibility.md](modules/visibility.md)).
- The identifier is the test's identity as a tool reports it. It need not be
  unique across a module — two tests MAY share a name, and a tool tells them
  apart by their positions — but it is never a name in the value namespace
  ([testing.md](testing.md#the-name)).
- The marker MAY carry one option group. The options in this change are
  `should_fail`, `should_panic` (optionally with a message), `skip`
  (optionally with a reason), and `timeout` (with milliseconds). The absent
  group means a normal test.
- The group is a non-empty comma-separated list; a trailing comma is allowed,
  and options may appear in any order. At most one expectation and at most one
  `timeout` may appear. `skip` MUST be the only option in its group.
  Combining `skip` with an expectation or `timeout` is `E0110`.
- An empty group, a duplicate option, two different expectations, a message on
  `should_fail`, a non-positive or non-integer timeout, or any other option
  name is not a valid `TestDecl` and is `E0110` when it parses as an ordinary
  attribute. This includes the deferred names `category`, `focus`, `retry`,
  and `threads`.
- A positional string, numeral, float, or boolean parses as an ordinary
  attribute and is `E0110`; a negative number is not grammatical under
  `AttributeValue` and remains a syntax error.
- The optional `async` marker says the body suspends; it is spelled exactly
  as `async` is spelled on a function, and it follows the attribute
  (`#[test] async fn` or `#[test(should_fail)] async fn`, never
  `async #[test] fn`).

```xulo
#[test]
fn test_parse() {
  assert(parse_ok(), "parses")
}

#[test]
fn absent_key_is_null() {
  let m = { a: 1 }
  assert(m.get("z") == null, "absent key is null")
}

#[test]
async fn delayed_task_finishes() {
  await Time::sleep(1)
}
```

The examples write `#[test]` on its own line directly above `fn`, and that
placement is **RECOMMENDED**: the attribute marks the whole declaration and
reads better apart from the signature. The grammar is newline-insensitive
([lexical-structure.md](lexical-structure.md)), so `#[test] fn name() { … }` on
one line is equally well-formed.

## Checking

- The body is checked exactly as a function body is checked: every rule that
  applies to the body of `fn(): unit` applies unchanged — types,
  exhaustiveness, mutability, the statement-value rule, and the diagnostics
  they carry. The body is checked whether or not any tool ever runs it, so a
  file whose tests do not check is rejected like any other file.
- The body is the body of an implicit nullary function returning `unit`: it
  declares no return type, `return` alone is accepted, and `return e` checks
  `e` against `unit`.
- The body sees the module exactly as a function body does: the file's
  `use` bindings and its module-level declarations, under the same scope and
  forward-reference rules
  ([modules/source-files.md](modules/source-files.md)).

## The name

A `#[test]` function's identifier is declared but binds no value:

- Any reference to it — a call, a value use, a path, or any other mention —
  is a compile-time error (`E0107`)
  ([type-system/errors.md](type-system/errors.md)). This includes a
  self-reference inside the test's own body: a recursive check is written as
  an ordinary helper function, which the test calls.
- The declaration is still well-formed without any reference: the name
  exists for the tool's discovery and reporting, and for nothing else.
- The restriction is the declaration's inertness in one rule — no execution
  of the program can reach a test body, because no reference to the name
  exists to reach it with.

## The `async` marker

- Without the marker, the body is an ordinary synchronous block: `await` in
  it is a compile-time error (`E0501`)
  ([type-system/errors.md](type-system/errors.md)).
- With the marker, the body is an `async` context exactly as the body of
  `async fn(): unit` is: `await` is legal, and when a tool runs the test the
  body starts as its own task.

## Options

A test declaration may state what it expects and how it is run:

| Marker | Meaning |
|--------|---------|
| `#[test]` | the body completes normally |
| `#[test(should_fail)]` | the body fails for any reason |
| `#[test(should_panic)]` | the body stops by a `panic` or a runtime failure |
| `#[test(should_panic = "text")]` | it stops by a `panic` or runtime failure whose message contains `text` |
| `#[test(skip)]` | the test is not executed |
| `#[test(skip = "reason")]` | the test is not executed; `reason` is reported |
| `#[test(timeout = 100)]` | the run must finish within 100 milliseconds |

`timeout` is not a mode: it may be combined with one expectation, in either
order. `skip` is not a mode either: it prevents the run, so it is alone.

```xulo
#[test(should_fail)]
fn rejected_input_fails() {
  assert(parse("") == null, "empty input is rejected")
}

#[test(should_panic)]
fn division_by_zero_panics() {
  divide(1, 0)
}

#[test(should_panic = "division by zero")]
fn division_by_zero_message() {
  divide(1, 0)
}

#[test(should_panic, timeout = 100)]
fn panic_arrives_quickly() {
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

### Expectations

- **`should_fail`** is the general inverse of a normal test. It passes when
  the body does not complete normally: a failed assertion, an explicit
  `panic`, a runtime failure, a cancelled `async` task, or a timeout all
  count. It fails when the body completes normally.
- **`should_panic`** is stricter. It passes only when the body stops by a
  `panic` or a runtime failure — a runtime failure is a panic raised by the
  implementation ([error-handling.md](error-handling.md)). It fails when the
  body completes normally, when an `async` test's task settles as cancelled,
  or when the deadline expires. It is not a synonym for `should_fail`.
- **`should_panic = "text"`** adds a message condition. For `panic(message)`,
  the failure message is the evaluated `message`. For a runtime failure, it is
  the implementation's description of that failure; that text is
  implementation-defined and documented
  ([memory-and-runtime.md](memory-and-runtime.md#implementation-defined)).
  The expected text is the cooked string literal; the failure message matches
  when it contains that text as a case-sensitive substring. An empty string
  matches every message.
- Exactly one expectation is allowed. `should_fail` takes no message; a group
  such as `should_fail = "text"` is `E0110`.
- A test body returns `unit`; a returned `Result` is data, not a test failure.
  `should_fail` is defined over the body outcome, so a future test form that
  treats a returned `Err` as a failure would extend it without a new
  expectation; `should_panic` would remain panic-only.

### Skip

- `skip` means the test is not executed. It is reported as skipped, with the
  reason when one is written; it is neither a pass nor a failure.
- The body is still checked normally, the name is still not referenceable
  (`E0107`), and `skip` does not suppress compile-time errors.
- `skip` is the only option in its group. `#[test(skip, should_fail)]`,
  `#[test(skip, timeout = 100)]`, and a duplicate `skip` are `E0110`.
- `skip = "reason"` carries a string for the report. The reason has no effect
  on checking or on any outcome; an empty reason is allowed.

### Timeout

- `timeout = N` sets a per-test wall-clock deadline of `N` milliseconds. `N` is an
  integer literal greater than zero; zero is `E0110` and a non-integer value
  is either `E0110` (when it parses as an ordinary named argument) or a syntax
  error (when it does not).
- The deadline starts when the body begins executing and includes time spent
  in `await` and ordinary execution. It does not include discovery, queueing,
  or tool overhead before the body starts.
- If the body settles before the deadline, the run has its normal outcome. If
  the deadline expires first, the outcome is `timedOut`; the tool stops
  waiting and reports the timeout. For an `async` test, the tool cancels the
  task. How a tool reclaims a synchronous body that ignores the deadline is
  outside the language, but it MUST stop waiting and MUST NOT later report a
  second outcome for that run.
- A timeout is a failure, not a panic. It passes `should_fail` and fails
  `should_panic`, including `should_panic = "…"`. A test that must fail for a
  particular reason should use `should_panic` with a message rather than
  `should_fail` with a short timeout.

The modes are deliberately distinct even where they coincide today. A
cancelled `async` test is the clearest current difference: it passes
`should_fail` and fails `should_panic`. Message matching is available only on
`should_panic`, where the cause is already known to be a panic.

## Running

- **A test body is inert.** No execution of the program evaluates it: not
  the module's initializers, not `main`, and not any call — the name binds no
  value, so nothing can reach the body. Statements in a test body are
  reachable only from a tool running that test.
- A test run ends in one of four outcomes:
  - **completed** — the body runs to completion; for a test marked `async`,
    the task its body started settles as completed;
  - **panicked** — a `panic` or a runtime failure stops the body;
  - **cancelled** — for an `async` test, the task its body started settles as
    cancelled while the test is running; if a configured deadline expires
    first, the outcome is `timedOut` instead;
  - **timedOut** — a configured `timeout` deadline expires before the body
    settles.
- A skipped test does not run and has no outcome; the tool reports it as
  skipped.
- The declared expectation selects the passing outcome:

  | Mode | Passes on | Fails on |
  |------|-----------|----------|
  | normal | completed | panicked, cancelled, timedOut |
  | `should_fail` | panicked, cancelled, timedOut | completed |
  | `should_panic` | panicked | completed, cancelled, timedOut |
  | `should_panic = "text"` | panicked with a matching message | completed, cancelled, timedOut, non-matching panic |

  Every other outcome is a failure. Without `timeout`, `timedOut` cannot
  occur.
- `panic` and runtime failures behave inside a test body as they do anywhere
  else: they stop the execution that reached them — here, the running of
  that test ([error-handling.md](error-handling.md)).
- Except for `skip` and the deadline outcome fixed here, everything else
  around a run — discovery, selection of non-skipped tests, order, isolation,
  report format, and whether a run continues after a failure — belongs to the
  tool and is outside this specification.

## Macros

A macro is tested at three layers. The language supplies the first and the
third; the second is a tooling contract and has no source syntax.

### 1. Implementation tests

A macro implementation function is an ordinary function, so a `#[test]` may
call its implementation identifier with constructed inputs:

```xulo
#[test]
fn serialize_derives_one_field() {
  let span = ast::Span { file: "test", start: 0, end: 0, line: 1, column: 1 }
  let input = ast::DeriveInput {
    target: ast::TypeInfo {
      kind: "struct",
      name: "User",
      fields: [
        ast::FieldInfo {
          name: "name", ty: "string", attrs: [], span: span
        }
      ],
      variants: [],
      generics: [],
      attrs: [],
      span: span
    }
  }
  match derive_serialize(input) {
    Result::Ok(output) => assert(output.items.len > 0, "derive emits code"),
    Result::Err(failure) => panic(failure.message)
  }
}
```

The input and output records are ordinary values
([memory-and-runtime.md](memory-and-runtime.md)); their fields are public, so a
test constructs them and inspects the result. The call is a runtime call: the
sandbox and the compile-time quota do not apply. A macro that calls an
`ast::call_*` entry point cannot be exercised at this layer, because the call
needs the compile-time registry; cover such a macro by the third layer.

### 2. Expansion tests

An expansion test compiles a fixture that invokes the macro and inspects the
expanded AST or the resulting diagnostic. This is a tooling feature: the
language defines no expansion assertion and no compile-fail syntax. A test
tool MAY expose an expanded-AST dump or a diagnostic matcher; that surface is
outside this specification. The contract the language fixes is the observable
one: the same source, compiler version, and structured input produce the same
expanded AST and the same diagnostics.

### 3. Integration tests

An ordinary `#[test]` exercises the code the macro generated:

```xulo
trait Serialize {
  fn serialize(self): Json
}

fn serialize_through_trait<T: Serialize>(value: T): Json {
  return value.serialize()
}

#[derive(Serialize)]
struct User { name: string, age: int }

#[test]
fn user_serializes() {
  let user = User(name: "Ada", age: 36)
  let json = serialize_through_trait(user)
  assert(json.entries.len == 0, "the derived impl is dispatched through the Serialize bound")
}
```

The generated `impl` participates in checking and dispatch exactly as a
written impl: `serialize_through_trait` resolves `value.serialize()` through
the `Serialize` bound to the generated method, so the integration test is an
ordinary test. A file whose macro invocation produces a compile-time
diagnostic does not compile, so a compile-fail test belongs to the tooling
layer, not to `#[test]`.

## Example

```xulo
#[test]
fn await_needs_the_async_marker() {
  await Time::sleep(1) // error: `await` outside of an `async` context (E0501)
}

#[test]
async fn await_with_the_async_marker() {
  await Time::sleep(1) // ok: the test passes when its task completes
}

#[test]
fn reference_is_forbidden() {
  assert(true, "runs only under a tool")
}

fn main() {
  reference_is_forbidden() // error: `reference_is_forbidden` is a test function and cannot be referenced (E0107)
}
```

The first pair differs only in the marker: the first test is rejected at check
time (`E0501`), the second is accepted. The last test is well-formed on its
own; the call in `main` is rejected (`E0107`), and the test body is still
checked and still runs only under a tool.
