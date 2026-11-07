# Tests — Delta: Macro Tests

> **Delta** — adds a “Macros” section to `testing.md`. The declaration,
> checking, name, `async`, and running rules for ordinary tests are unchanged.

## Macros

A macro is tested at three layers. The language supplies the first and the
third; the second is a tooling contract and has no source syntax.

### 1. Implementation tests

A `#[macro]` function is an ordinary function, so a `#[test]` may call its
implementation identifier with constructed inputs:

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
#[derive(Serialize)]
struct User { name: string, age: int }

#[test]
fn user_serializes() {
  let user = User(name: "Ada", age: 36)
  let json = user.serialize()
  assert(json.entries.len == 0, "the generated serialize impl returns an empty entry list")
}
```

The generated `impl` participates in checking and dispatch exactly as a
written impl, so the integration test is an ordinary test. A file whose macro
invocation produces a compile-time diagnostic does not compile, so a
compile-fail test belongs to the tooling layer, not to `#[test]`.
