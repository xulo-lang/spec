# Tests — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the opening sentence, the **Declaration** section,
> the **The name** opening sentence, and the **Example** block of
> `testing.md` with the lowercase bracket spelling. The checking, `async`,
> discovery, and pass/fail rules are unchanged.

## Declaration

The form is `TestDecl` in the grammar ([grammar.md](grammar.md)):

```text
TestDecl = "#" "[" "test" [ "(" [ StringLiteral ] ")" ] "]" [ "async" ] "fn"
           Identifier "(" ")" Block ;
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
  apart by their positions or their titles — but it is never a name in the
  value namespace ([The name](#the-name) below).
- An optional parenthesized string inside the attribute brackets —
  `#[test("parse test")]` — is the test's **title**: metadata the declaration
  carries for a tool's reporting. It is not a name in any namespace, it
  binds nothing, and it need not be unique. An empty group `#[test()]` and
  the bare form `#[test]` both mean no title; the bare form is
  **RECOMMENDED** when no title is wanted.
- The optional `async` marker says the body suspends; it is spelled exactly
  as `async` is spelled on a function, and it follows the attribute
  (`#[test("…")] async fn`, never `async #[test] fn`).

```xulo
#[test("parse test")]
fn test_parse() {
  assert(parse_ok(), "parses")
}

#[test]
fn absent_key_is_null() {
  let m = { a: 1 }
  assert(m.get("z") == null, "absent key is null")
}

#[test()]
async fn delayed_task_finishes() {
  await Time::sleep(1)
}
```

The examples write `#[test]` on its own line directly above `fn`, and that
placement is **RECOMMENDED**: the attribute marks the whole declaration and
reads better apart from the signature. The grammar is newline-insensitive
([lexical-structure.md](lexical-structure.md)), so `#[test] fn name() { … }`
on one line is equally well-formed.

## Chapter introduction and name

Replace the opening sentence with:

A program's tests are declarations in its source: the `#[test]` form marks a
function to be checked with the rest of the file and to be executed only by a
test tool.

Replace the `The name` opening sentence with:

A `#[test]` function's identifier is declared but binds no value:

## Example

Replace the example block with:

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
  reference_is_forbidden() // error: a test function cannot be referenced (E0107)
}
```
