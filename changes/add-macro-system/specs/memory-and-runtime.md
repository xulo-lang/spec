# Memory and Runtime — Delta: Compile-Time Macro Values

> **Delta** — adds a “Compile-time macro values” section to
> `memory-and-runtime.md`, defining the `ast::` records a macro receives and
> returns. The value semantics, ownership, borrowing, runtime failure, and
> exit-status rules are unchanged.

## Compile-time macro values

The macro interface is a set of ordinary built-in values, all members of the
`ast` namespace ([builtins/prelude.md](builtins/prelude.md)). They are
available at runtime — a test can construct them and call a macro function —
and at compile time, where the compiler executes the same function. None of them is
a reserved identifier: a program reaches them through `ast::`.

```typescript
struct ast::Span {
  file: string;
  start: int;      // byte offset, inclusive
  end: int;        // byte offset, exclusive
  line: int;       // 1-based
  column: int;     // 1-based
}

struct ast::AstNode {
  kind: string;    // the AST node kind, e.g. "StructDeclaration"
  span: ast::Span;
}

enum ast::Value {
  Unit,
  Bool(bool),
  Int(int),
  Float(float),
  String(string),
  List(List<ast::Value>),
  Map(Map<string, ast::Value>),
  AstNode(ast::AstNode),
}

struct ast::MacroFailure {
  message: string;
  span: ast::Span?;
}
```

`ast::AstNode` is opaque beyond `kind` and `span`: the structure of a node is
read through `ast::fields_of` and `ast::field` and written through
`ast::quote!` or `ast::node`
([builtins/context.md](builtins/context.md)). `ast::Span` is a copyable value;
a macro never creates a span out of nothing, but it may copy one from its
input or from `ast::current_span()`. `ast::Value` is a recursive enum; the
`List` and `Map` payloads provide the indirection, so the recursion is the
permitted enum recursion of
[`types/enums.md`](types/enums.md).

### Input records

```typescript
struct ast::DeriveInput {
  target: ast::TypeInfo;
}

struct ast::TypeInfo {
  kind: "struct" | "enum";
  name: string;
  fields: List<ast::FieldInfo>;
  variants: List<ast::VariantInfo>;
  generics: List<ast::GenericParam>;
  attrs: List<ast::AttributeInfo>;
  span: ast::Span;
}

struct ast::FieldInfo {
  name: string;
  ty: string;                    // the written type, in source form
  attrs: List<ast::AttributeInfo>;
  span: ast::Span;
}

struct ast::VariantInfo {
  name: string;
  fields: List<ast::FieldInfo>;  // positional payload slots, in order
  attrs: List<ast::AttributeInfo>;
  span: ast::Span;
}

struct ast::GenericParam {
  kind: "type" | "const";
  name: string;
  bounds: List<string>;          // empty for a const parameter
  ty: string?;                   // "int" for a const parameter, null for a type parameter
  span: ast::Span;
}

struct ast::AttributeInfo {
  name: string;
  args: List<ast::AttributeArgInfo>;
  span: ast::Span;
}

struct ast::AttributeArgInfo {
  name: string?;                 // null for a positional argument
  value: ast::Value;
}

struct ast::AttributeInput {
  args: List<ast::Value>;        // positional arguments, in source order
  named_args: Map<string, ast::Value>;  // name = value arguments
  target: ast::DeclarationInfo;
}

struct ast::DeclarationInfo {
  kind: "fn" | "struct" | "enum" | "trait" | "impl"
      | "type" | "let" | "const";
  name: string;
  node: ast::AstNode;            // the original declaration, for the output item
  params: List<ast::ParamInfo>;  // empty unless kind = "fn"
  return_type: string?;          // null unless kind = "fn"
  attrs: List<ast::AttributeInfo>;
  trait_ref: string?;            // kind = "impl" only
  self_type: string?;            // kind = "impl" only
  span: ast::Span;
}

struct ast::ParamInfo {
  name: string;
  ty: string;
  mode: "value" | "mut" | "shared";
  has_default: bool;
  span: ast::Span;
}

struct ast::FunctionInput {
  args: List<ast::Value>;        // a literal is a scalar; another expression is an AstNode
  target: ast::ExpressionInfo;
}

struct ast::ExpressionInfo {
  kind: string;                  // "macro_call" for the invocation itself
  text: string;                  // the written expression
  node: ast::AstNode;
  span: ast::Span;
}
```

`ast::DeclarationInfo.node` is the original declaration node. An attribute
macro returns it as `ast::AttributeOutput.item` when it does not replace the
declaration; without the node, the mandatory `item` field could not be
produced. `ast::FunctionInput.target.node` is the `MacroCallExpr` itself, so a
macro may return it unchanged or inspect it.

`ast::DeclarationInfo.kind` has exactly the eight values the grammar admits
for an attribute host. A `"type"` kind corresponds to the `TypeAlias` AST
node, `"let"` to `LetDeclaration`, and `"const"` to `ConstDeclaration`.
`params` is empty and `return_type` is `null` for every kind except `"fn"`;
`trait_ref` and `self_type` are non-null only for `"impl"`.

### Output records

```typescript
struct ast::DeriveOutput {
  items: List<ast::AstNode>;
}

struct ast::AttributeOutput {
  item: ast::AstNode;            // required: the replacement declaration
  extras: List<ast::AstNode>;
}

struct ast::FunctionOutput {
  expr: ast::AstNode;
}
```

`ast::AttributeOutput.item` is not optional: an attribute macro replaces or
returns the decorated declaration, and never deletes it
([type-system/checking-rules.md](type-system/checking-rules.md)).

### Functions in the `ast` namespace

```typescript
// ast::quote! { … } : List<ast::AstNode>, see calls.md
fn ast::call_derive(name: string, input: ast::DeriveInput | ast::Value)
    : Result<ast::DeriveOutput, ast::MacroFailure>
fn ast::call_attribute(name: string, input: ast::AttributeInput | ast::Value)
    : Result<ast::AttributeOutput, ast::MacroFailure>
fn ast::call_function(name: string, input: ast::FunctionInput | ast::Value)
    : Result<ast::FunctionOutput, ast::MacroFailure>
fn ast::node(kind: string, fields: Map<string, ast::Value>): ast::AstNode
fn ast::field(node: ast::AstNode, name: string): ast::Value?
fn ast::current_span(): ast::Span
fn ast::span_of(node: ast::AstNode): ast::Span
fn ast::compile_date(): string
fn ast::lookup_type(name: string): ast::TypeInfo?
fn ast::lookup_function(name: string): string?
fn ast::lookup_variable(name: string): string?
fn ast::fields_of(name: string): List<ast::FieldInfo>
```

The context and call functions are specified in
[builtins/context.md](builtins/context.md); the sandbox rule for
`compile_date` is in [builtins/sandbox.md](builtins/sandbox.md). A macro
function is an ordinary function, so these values obey the ordinary value,
ownership, and borrowing rules. The implementation's internal representation
may be richer; the public `ast::Value` is the subset a macro sees.
