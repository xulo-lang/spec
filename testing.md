# Tests

A program's tests are declarations in its source: the `#Test` form marks a
function to be checked with the rest of the file and to be executed only by a
test tool. This chapter specifies the declaration's form, when its body is
checked, what the `async` marker adds, and what it means for a test to pass or
fail. Which tests a tool discovers, the order it runs them, what each run is
isolated from, and how a result is reported are properties of the tool, not of
this specification ([README §1.1](README.md)).

## Declaration

The form is `TestDecl` in the grammar ([grammar.md](grammar.md)):

```text
TestDecl = "#Test" [ "async" ] "fn" Identifier "(" ")" Block ;
```

- A `#Test` function appears at file scope, among a module's top-level
  declarations ([modules/source-files.md](modules/source-files.md)); it is
  not a statement and appears nowhere else.
- It carries no `pub`, no parameters, no return type, no generic parameters,
  and no `where` clause: the body is an implicit `fn(): Unit`, and the
  declaration is never part of an interface
  ([modules/visibility.md](modules/visibility.md)).
- The identifier is the test's identity as a tool reports it. It need not be
  unique across a module — two tests MAY share a name, and a tool tells them
  apart by their positions — but it is never a name in the value namespace
  ([The name](#the-name) below).
- The optional `async` marker says the body suspends; it is spelled exactly
  as `async` is spelled on a function, and it follows the attribute
  (`#Test async fn`, never `async #Test fn`).

```xulo
#Test
fn mapGetAnswersNull() {
  let m = { a: 1 }
  assert(map_get(m, "z") == null, "absent key is null")
}

#Test
async fn delayedTaskFinishes() {
  await Time::sleep(1)
}
```

The examples write `#Test` on its own line directly above `fn`, and that
placement is **RECOMMENDED**: the attribute marks the whole declaration and
reads better apart from the signature. The grammar is newline-insensitive
([lexical-structure.md](lexical-structure.md)), so `#Test fn name() { … }` on
one line is equally well-formed.

## Checking

- The body is checked exactly as a function body is checked: every rule that
  applies to the body of `fn(): Unit` applies unchanged — types,
  exhaustiveness, mutability, the statement-value rule, and the diagnostics
  they carry. The body is checked whether or not any tool ever runs it, so a
  file whose tests do not check is rejected like any other file.
- The body is the body of an implicit nullary function returning `Unit`: it
  declares no return type, `return` alone is accepted, and `return e` checks
  `e` against `Unit`.
- The body sees the module exactly as a function body does: the file's
  `use` bindings and its module-level declarations, under the same scope and
  forward-reference rules
  ([modules/source-files.md](modules/source-files.md)).

## The name

A `#Test` function's identifier is declared but binds no value:

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
  `async fn(): Unit` is: `await` is legal, and when a tool runs the test the
  body starts as its own task.

## Running

- **A test body is inert.** No execution of the program evaluates it: not
  the module's initializers, not `main`, and not any call — the name binds no
  value, so nothing can reach the body. Statements in a test body are
  reachable only from a tool running that test.
- A running test passes or fails by a criterion fixed here:
  - it **passes** when its body runs to completion — for a test marked
    `async`, when the task its body started settles as completed;
  - it **fails** when a `panic` or a runtime failure stops the body, or —
    for a test marked `async` — when that task settles as cancelled.
- `panic` and runtime failures behave inside a test body as they do anywhere
  else: they stop the execution that reached them — here, the running of
  that test ([error-handling.md](error-handling.md)).
- Everything else around a run — which tests execute, in what order, each in
  what isolation, what a tool reports, and whether a run continues after a
  failure — belongs to the tool and is outside this specification.

## Example

```xulo
#Test
fn awaitNeedsTheAsyncMarker() {
  await Time::sleep(1) // error: `await` outside of an `async` context (E0501)
}

#Test
async fn awaitWithTheAsyncMarker() {
  await Time::sleep(1) // ok: the test passes when its task completes
}

#Test
fn referenceIsForbidden() {
  assert(true, "runs only under a tool")
}

fn main() {
  referenceIsForbidden() // error: `referenceIsForbidden` is a test function and cannot be referenced (E0107)
}
```

The first pair differs only in the marker: the first test is rejected at check
time (`E0501`), the second is accepted. The last test is well-formed on its
own; the call in `main` is rejected (`E0107`), and the test body is still
checked and still runs only under a tool.
