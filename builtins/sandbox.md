# Macro Sandbox

Compile-time execution is sandboxed: a macro can read the program's
compile-time context and construct AST, and nothing else. The sandbox exists
so that compiling an untrusted package cannot read the user's files, open
sockets, or make the build depend on the wall clock.

## Allowed and forbidden

| Resource | Compile-time access |
|----------|---------------------|
| Filesystem | forbidden |
| Network | forbidden |
| Environment variables | forbidden |
| Processes | forbidden |
| Concurrency (`spawn`, `Task`, `lock`) | forbidden |
| Wall-clock time | forbidden |
| Randomness | forbidden |
| Compile-time context (types, functions, variables, fields, spans) | allowed, read-only |
| AST construction and `ast::quote!` | allowed |
| `ast::compile_date()` | allowed |

The table is exhaustive for this version. A macro that reaches a forbidden
resource fails the expansion with `E1007`; the diagnostic names the resource
and lists the allowed set.

## `ast::compile_date`

```typescript
fn ast::compile_date(): string
```

`ast::compile_date` returns the compilation date in `YYYY-MM-DD` form, in UTC,
at day granularity. It is the only time source. A macro MUST NOT use it to
make a build depend on a finer clock; there is no such clock. The expansion
cache includes the compile date in its key
([cache.md](cache.md)), so a change at midnight invalidates the affected
expansions.

## Runtime versus compile time

The sandbox applies only when the compiler executes a `#[macro]` function during
expansion. The same function is an ordinary function at runtime: a `#[test]` or
ordinary call runs it without the sandbox, and filesystem or clock access in
that call is ordinary program behaviour. A macro that succeeds in a runtime
test but uses the filesystem still fails as a macro with `E1007`.

This split is deliberate. It keeps the implementation function testable with
the language's own tests without weakening the compile-time boundary. The
contract a macro author can rely on is: **if the function can be expanded, it
used no forbidden resource**.

## Failure

A sandbox violation is a compile-time diagnostic:

```text
error[E1007]: macro `sql` cannot access the filesystem
  --> macros/sql.xulo:12:3
   |
12 |   let text = File::read(path)
   |   ^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: compile-time macros may not access the filesystem, network,
           environment, processes, concurrency, wall-clock time, or randomness
   = note: the invocation is at main.xulo:8:13
```

The primary span is the source operation in the macro definition that attempted
the access; the span chain reaches the invocation
([errors.md](errors.md)).
