# Type Errors and Diagnostics — Delta: Macro Diagnostics

> **Delta** — adds the `E1001`–`E1099` family row to the error-code ranges and
> the `E1001`–`E1012` rows to a new “Macro errors” section of
> `type-system/errors.md`. The rest of the chapter is unchanged.

## Error code ranges

Add this row after the `E0901`–`E0999` row:

| `E1001`–`E1099` | macros, compile-time execution, expansion | [builtins/errors.md](../builtins/errors.md) |

## Macro errors

A macro expansion has three error sources, and each source owns a distinct
type. The compiler never has to infer which source a diagnostic came from.

```typescript
// Authored by the macro; also the `E` of its Result.
struct ast::MacroFailure {
  message: string;
  span: ast::Span?;
}

// Raised by compile-time execution.
enum ExecutionError {
  QuotaExceeded { quota: string, limit: int, used: int },        // E1001
  ConversionFailed { expected: string, actual: string },          // E1002
  RecursiveMacroCall { chain: List<QualifiedMacroName> },          // E1003
  SandboxViolation { resource: string },                           // E1007
  MacroPanic { message: string },                                  // E1012
}

// Raised by the expansion loop.
enum ExpansionError {
  MaxRoundsExceeded { rounds: int, last_macro: QualifiedMacroName }, // E1004
  MacroNotFound { name: QualifiedMacroName },                        // E1005
  SignatureMismatch { expected: string, actual: string },            // E1006
  LocalNameUnresolved { name: string },                              // E1009
  InvalidMacroDefinition { reason: string },                         // E1010
  MalformedQuote { reason: string },                                 // E1011
}

// A fully qualified macro name: module path plus macro name.
struct QualifiedMacroName {
  module: List<string>;
  name: string;
}
```

`ast::MacroFailure` is an ordinary built-in record
([memory-and-runtime.md](../memory-and-runtime.md)); the other two enums are
compiler-internal and never appear in user code.

`E1001`–`E1003`, `E1007`, and `E1012` belong to `ExecutionError`; `E1004`–`E1006`,
`E1009`, `E1010`, and `E1011` belong to `ExpansionError`; `E1008` is the
`ast::MacroFailure` the macro author returns. `MaxRoundsExceeded` is an
expansion-loop error, not an execution error: rounds are a property of the
loop, not of one execution.

The runtime failures of [`../memory-and-runtime.md`](../memory-and-runtime.md)
— out-of-range index, division by zero, `panic` — are **not** given new codes
and are **not** checker diagnostics. A macro that triggers one is a compile-time
trap: compile-time execution stops the expansion and reports `E1012`, whose
primary span is the trapping source operation inside the macro definition and
whose span chain reaches the invocation. A runtime failure in ordinary program code remains a
panic with no `E` code.

### Table

| Code | Trigger | Example | Message shape | Specified in |
|------|---------|---------|---------------|--------------|
| `E1001` | a macro exceeds its step, memory, or call-depth quota | a derive that loops over every field recursively | ``macro `Serialize` exceeded its step quota`` | [builtins/quota.md](../builtins/quota.md) |
| `E1002` | an `ast::Value` input to an `ast::call_*` entry point cannot be converted to the target input record | `ast::call_derive("m::derive", ast::Value::AstNode(target))` where `target` is a function declaration | ``cannot convert `FnDeclaration` to `DeriveInput` `` | [builtins/execution.md](../builtins/execution.md) |
| `E1003` | the compile-time call chain already includes the macro | `m::a` → `m::b` → `m::a` | ``recursive macro call: `m::a` → `m::b` → `m::a` `` | [builtins/execution.md](../builtins/execution.md) |
| `E1004` | expansion exceeds 100 rounds | `recursive!(x)` generating `recursive!(x)` | ``macro expansion exceeded maximum rounds (100)`` | [type-system/checking-rules.md](checking-rules.md) |
| `E1005` | a derive target, a function macro name, or an `ast::call_*` name resolves to no macro | `ast::call_derive("m::missing", input)` with no `m::missing` macro in scope | ``cannot find macro `m::missing` `` | [type-system/checking-rules.md](checking-rules.md) |
| `E1006` | a macro is invoked in the wrong kind or its input/output shape does not match its declaration, including an `ast::call_*` entry point naming a macro of another kind | `#route(…)` where `route` is a function macro | ```route` is a function macro, not an attribute macro`` | [type-system/checking-rules.md](checking-rules.md) |
| `E1007` | a macro uses a resource the sandbox forbids | reading a file, opening a socket, reading the clock | ``macro `sql` cannot access the filesystem`` | [builtins/sandbox.md](../builtins/sandbox.md) |
| `E1008` | the macro returns `Result::Err(ast::MacroFailure { … })` | `#route("/v2/users")` | ``macro `route` reported an error`` | [builtins/errors.md](../builtins/errors.md) |
| `E1009` | a name in a function macro argument is not in the pre-expansion scope | `sql!(user_name)` with no `user_name` declared | ``cannot resolve `user_name` in macro arguments`` | [expressions/calls.md](../expressions/calls.md) |
| `E1010` | a `#Macro` declaration is malformed, duplicate, or collides with a built-in | `#Macro(kind = "derive")` with no `target` | ``invalid macro definition: derive macro requires `target` `` | [type-system/checking-rules.md](checking-rules.md) |
| `E1011` | a quote body does not parse after substitution, or a hole value cannot be inserted there | `${name}` where `name` is a list in an identifier position | ``quote body is not well-formed`` | [expressions/calls.md](../expressions/calls.md) |
| `E1012` | macro execution traps: index out of range, division by zero, or `panic` | `fields[99]` in a macro | ``macro `Serialize` trapped: index 99 out of range`` | [builtins/execution.md](../builtins/execution.md) |

`E1001` carries `= note: quota source: …` and the current usage; `E1003`
carries the call chain and each macro definition site; `E1005` carries the
macro namespace searched; `E1006` carries the expected kind or signature;
`E1007` carries the allowed-resource list; `E1008` carries the macro author's
message and the span chain; `E1009` carries the note that macro arguments are
resolved before expansion; `E1010` carries the offending key; `E1011` carries
the offending fragment; `E1012` carries the execution trap's panic message.
