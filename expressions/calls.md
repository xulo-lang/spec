# Calls

A call applies a function, method, closure, or component to arguments. The
call syntax is uniform — `callee(arguments)` — and the rules below define how
arguments are matched to parameters, in what order they are evaluated, and
what is an error.

## Call syntax

A call is a postfix operator: a callee followed by a parenthesized,
comma-separated argument list. A trailing comma is allowed, and an empty
argument list — `f()` — passes no arguments.

```xulo
greet("world")
math::max(a, b,)
```

```text
Call         = Callee '(' [ ArgumentList ] ')' ;
ArgumentList = Argument ( ',' Argument )* ','? ;
Argument     = Identifier ':' ( "$" Identifier | Expression ) (* named *)
             | "$" Identifier                              (* bound state *)
             | Expression                                  (* positional *) ;
```

For a declared `fn`, the number of arguments supplied MUST match the declared
parameter list once default and optional parameters are accounted for: every
parameter that is neither defaulted nor optional MUST receive an argument, and
no parameter MAY receive more than one argument. See
[`../functions.md`](../functions.md) for declarations.

## Positional arguments

Positional arguments are matched to parameters in declaration order. They are
evaluated strictly left-to-right, as specified in
[`README.md`](README.md) (§ Evaluation Order), and every argument is
evaluated exactly once:

```xulo
log("start", timestamp())
```

## Named arguments

A named argument writes the parameter's name followed by `:` and an
expression.

```xulo
Button(variant: "outline", label: "Submit")
```

Named arguments MAY be reordered relative to the parameter list. Each named
argument MUST name a parameter declared by the callee, and naming the same
parameter twice is an error (`E0208`). Positional arguments come first: once a named
argument appears, every following argument MUST be named as well. Mixing is
therefore one-directional:

```xulo
fn render(text: string, size: int, bold: bool): View {
  Text(text)
}

render("Hi", bold: true, size: 14)   // OK: positional, then named
render(text: "Hi", 14, bold: true)   // error: positional after named
```

## Binding arguments

In a component invocation an argument MAY be the binding form `$name`, labeled
(`value: $name`) or bare:

```xulo
Input(value: $name)
```

`$name` is not an expression: it passes a binding — the state cell of the
named variable — rather than a value, it is legal only in an argument position
of a component invocation, and the name MUST be declared by `@State` or
`@Store` in the enclosing component body. The mechanism and its errors are
specified in [`../components/binding.md`](../components/binding.md).

## Default parameters

A parameter MAY declare a default with `=`.

```xulo
fn greet(name: string = "stranger"): string {
  "Hello, " + name
}
```

A default value is evaluated at each call that omits the argument, not once at
declaration, so a default may depend on values computed at call time. The
parameter's type MUST admit the default value: `name: string = 42` is a
compile-time error (`E0228`).

Any parameter MAY have a default. An omitted argument MUST be supplied by name
or MUST be trailing in the positional list: a positional call MAY omit only a
suffix of the parameters, and every parameter in that suffix MUST have a
default (or be optional, below). A call that leaves a required parameter
unfilled, or that fills a later parameter while skipping an earlier one, is an
error.

## Optional parameters

A parameter whose type is optional (`T?`) MAY be omitted at the call site, in
which case it is bound to `null`.

```xulo
fn greet(name: string?): string {
  if name != null { "Hello, " + name } else { "Hello, stranger" }
}

greet()      // name is null
greet("Ada")
```

An optional parameter without a default accepts `null` explicitly as well, so
`greet(null)` and `greet()` are equivalent. Optional and defaulted parameters
follow the same positional-trailing or named rule stated above.

## Method calls

A method call writes `receiver.method(arguments)`.

```xulo
r.area()
r.is_square()
```

Resolution proceeds in this order: first the methods declared in `impl` blocks
for the receiver's own type, then methods provided by trait implementations
that are in scope at the call site (see
[`../types/traits.md`](../types/traits.md)). The explicit form
`Trait.method(receiver)` is always available and is required when the
receiver's type is a generic parameter bounded by a trait. The receiver is
supplied to the declared `self` parameter, which is not written at the call
site; argument matching, evaluation order, and arity otherwise follow the
rules above. A method call whose receiver is `null` is written with optional
chaining, `receiver?.method()`, which evaluates to `null` instead of
evaluating the call (see [`path-and-access.md`](path-and-access.md)).

## Generic calls

Type arguments of a generic function are inferred at the call site from the
actual arguments and from the expected type. A call MAY also write them
explicitly, between the callee and the argument list, and the written list
takes the place of inference for that call. A callee that declares const
parameters MUST be given a written list — `count<3>()` — because const
arguments are never inferred (`E0816`). The written list covers type and
const parameters together: its length is `E0808`, and a type/const kind
mismatch is `E0813`:

```xulo
fn first<T>(items: List<T>): T {
  items[0]
}

let n: int = first([1, 2, 3])    // T is int, inferred
let t = first<int>([1, 2, 3])    // T is int, written
let s = first(["a", "b"])        // T is string, inferred

fn count<const N: int>(): int { N }

let c = count<3>()          // N is 3, a const argument is always written
```

Whether a `<` after a callee begins such a list is decided by a purely
syntactic rule. It does so only when both conditions hold:

1. the material between that `<` and the matching `>` parses as a
   comma-separated list of types — the empty material counts as a zero-length
   list, so `f<>(…)` reaches the arity diagnostic instead of falling back to a
   comparison; and
2. the token immediately after the matching `>` is `(`.

If either condition fails, the `<` is the relational operator and the
expression is a comparison. The scan for the matching `>` counts angle depth
and skips over `(` `)`, `[` `]`, and `{` `}` groups. The decision MUST NOT
consult name resolution, and backtracking from the type-argument reading to
the comparison reading is allowed whenever the call parse does not complete.
The rule is scoped to an identifier in call position: a callee reached through
a subscript or any other postfix that does not end in an identifier takes no
list, and a call through a value of function type has no generic parameters and
accepts none. `>>` and `>=` are single tokens, so the nested-bracket spacing
rule applies — `first<Map<string, int> >(m)` is well-formed and
`first<Map<string, int>>(m)` is not. The production is given in
[grammar.md](../grammar.md).

Arity, bounds, substitution without inference, and the interaction with an
expected type are specified in [`../types/generics.md`](../types/generics.md);
the diagnostics are `E0807` (a callee that accepts no list), `E0808` (a list
of the wrong length), and `E0801` (a written argument that satisfies no
bound). A call that writes no list is checked by inference, exactly as
specified in that chapter.

## Closures as callees

Any expression of function type may be called, including a variable holding a
closure, a list element, a subscript, and the result of another call.

```xulo
double(21)
xs[0](10)
get_fn()(x)
(f)(5)
(fn(x: int): int { x })(1)
```

The callee is evaluated first, then the arguments left-to-right. A call made
through a value of function type (`fn(...)`) accepts positional arguments
only, and the count MUST match exactly — named arguments are not available
through function values, only through declared functions and methods. See
[`closures.md`](closures.md).

## Component calls

A component call looks like a function call whose declared return type is
`View`, and may be followed by a trailing block of children.

```xulo
Text("Hi")
Button("Go", variant: "primary") {
  Text("Confirm")
}
```

The first positional argument is conventionally the component's content or
label, named arguments are attributes, and a trailing block becomes the
component's children rather than an argument. Component calls follow the same
argument rules as function calls; the component layer — attributes, blocks,
and `$` binding (above) — is specified in
[`../components/view-syntax.md`](../components/view-syntax.md) and
[`../components/binding.md`](../components/binding.md).

## Macro calls

A function macro is invoked as a path followed by `!` and a parenthesized
expression list:

```xulo
let table = sql!("SELECT * FROM users WHERE id = id")
let formatted = m::json!(value)
let empty = build!()
```

```text
MacroCall         = Path "!" "(" [ MacroArgumentList ] ")" ;
MacroArgumentList = Expression { "," Expression } [ "," ] ;
```

A macro call is not a function call:

- The path names a macro in the macro namespace, not a value or a function.
  A missing macro is `E1005`; a macro of another kind is `E1006`
  ([type-system/checking-rules.md](../type-system/checking-rules.md)).
- There are no named arguments, no `$` binding arguments, and no explicit
  type-argument list: the macro receives its input record, not a parameter
  list.
- The arguments are not evaluated. A literal argument becomes an
  `ast::Value` (a string, `int`, `float`, or `bool`); every other expression
  becomes an `ast::AstNode`. The macro decides what to do with the node.
- Before the macro runs, every name mentioned by an argument MUST resolve in
  the pre-expansion scope. A name that resolves only after expansion — a name
  generated by another macro — is `E1009`, with the note that macro arguments
  are resolved before expansion. The repair is to let one function macro
  generate the whole region, or to use an `ast::call_*` entry point inside
  the macro.

The expanded expression replaces the macro call before type checking, so its
type is the ordinary type of the generated expression. A macro whose output
is not an expression is `E1006`. A macro call written inside another macro
argument is expanded before the outer call, in source order.

A macro name is not a value: `let f = sql` is `E0101`. The implementation
function's own name is an ordinary value if it is `pub` and imported.

## Quasi-quote

`ast::quote! { … }` builds a list of AST nodes from source:

```xulo
#[macro(kind = "derive", target = Serialize)]
fn derive_serialize(
  input: ast::DeriveInput
): Result<ast::DeriveOutput, ast::MacroFailure> {
  let name = input.target.name
  return Result::Ok(ast::DeriveOutput {
    items: ast::quote! {
      impl Serialize for ${name} {
        fn serialize(self): Json { Json { entries: [] } }
      }
    }
  })
}
```

```text
QuoteCall  = "ast" "::" "quote" "!" "{" { QuotePiece } "}" ;
QuotePiece = QuoteHole
           | ? any balanced token sequence that does not begin with "${"
               and does not contain an unmatched "}" ? ;
QuoteHole  = "${" Expression "}" ;
```

The body is a token fragment, not a block: it may hold declarations,
statements, or a partial declaration, and its braces are balanced. The
compiler substitutes each hole and parses the result as a sequence of
declarations and statements. The result is a `List<ast::AstNode>`.

A hole value is converted to tokens by this table:

| Value | Inserted as |
|-------|-------------|
| `string` | the string's characters, lexed as source — usually an identifier, a path, or a type |
| `int`, `float` | a numeric literal |
| `bool` | `true` or `false` |
| `ast::AstNode` | the node's own tokens |
| `List<ast::AstNode>` | the concatenation of the nodes' tokens |

A string is lexed as source because the common case is a name or a type, not a
string literal. To emit a string literal, the macro writes the quotes in the
quoted body or constructs the literal node explicitly. A hole whose value
cannot be inserted in its position, or whose substituted text does not parse,
is `E1011`. A hole in an identifier position, a type position, an expression
position, and a pattern position inserts the tokens at that position; the
compiler does not rewrite the value to fit the grammar.

`ast::quote!` is the only brace-delimited macro form. A user function macro is
called with parentheses only.

## Macro-to-macro calls

A macro function can invoke another macro during compile-time execution:

```xulo
let output = ast::call_attribute("m::helper", input)?
let item = output.item
```

`ast::call_derive`, `ast::call_attribute`, and `ast::call_function` each take a
fully qualified macro name (a string) and either the matching input record or an
`ast::Value` target node. They return the matching output record, so no output
conversion is needed. Each uses the same macro-call protocol with the same
input ([builtins/execution.md](../builtins/execution.md)).

The target must be the same kind as the entry point; naming a macro of another
kind is `E1006`. If the input is an `ast::Value`, the compiler converts it to
the input record and reports `E1002` when the conversion fails:

| Input value | Converted to |
|-------------|--------------|
| `AstNode` of a struct or enum declaration | `ast::DeriveInput` for `ast::call_derive` |
| `AstNode` of a declaration | `ast::AttributeInput` for `ast::call_attribute`, with empty `args` and `named_args` |
| `AstNode` of an expression | `ast::FunctionInput` for `ast::call_function`, with empty `args` |
| any other `ast::Value` | `E1002` |

Record fields are not read from the opaque `AstNode` surface, which exposes only
`kind` and `span`. The compiler computes them from the node's original
pre-expansion declaration or expression, using the same routines that build a
direct invocation's input: `TypeInfo` for a derive target, `DeclarationInfo`
for an attribute target, and `ExpressionInfo` for a function target (`text` is
the node's source text). A node with no original declaration or expression —
one built with `ast::node`, or a quoted fragment — cannot provide these fields
and is `E1002`.

A typed input record is used as is; the table applies only to an
`ast::Value`. A node shorthand has empty argument lists, so a macro that needs
arguments passes its own typed input record — for example, an attribute macro
passes the `ast::AttributeInput` it already holds.

The call runs in the current budget and call chain (`E1001`, `E1003`). The
fully qualified name is resolved in the macro namespace; an unqualified name is
resolved in the calling macro's definition module. At runtime — when the same
function is called as an ordinary function — the call entry points have no
macro registry and trap; they exist for compile-time execution. See
[builtins/context.md](../builtins/context.md).

## Recursion and forward references

A call requires its callee to be declared. Recursion is unrestricted: a
function may call itself. Module-level functions may also forward-reference
one another, so mutual recursion between functions in the same module is
well-formed without any ordering constraint (see
[`../names.md`](../names.md)). A call to an undeclared or private callee
outside its module is a compile-time error (`E0101`, `E0602`).

## Evaluation errors

Calling a value that is not of function type is a compile-time error (`E0204`); there
is no dynamic dispatch to an arbitrary value and no truthiness that would make
a non-function callable. Arguments MUST match the declared parameter types,
and a mismatch — including a missing required argument, an unknown named
argument, a duplicate argument, or an argument of an unrelated type — is
reported as a compile-time error (`E0202`, `E0207`, `E0208`)
(see [`../type-system/errors.md`](../type-system/errors.md)). Failures that cannot
be detected statically, such as indexing past the end of a list inside the
call, are runtime errors (see [`../memory-and-runtime.md`](../memory-and-runtime.md)).
