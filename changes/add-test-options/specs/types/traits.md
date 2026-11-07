# Traits — Delta: Test Options

> **Delta** — replaces the `### Test` entry of `types/traits.md`. The rest of
> the Attributes section is unchanged.

## Test

`#[test]` marks a function as a test. It is spelled only by the test form —
`#[test]`, `#[test(should_fail)]`, `#[test(should_panic)]`,
`#[test(should_panic = "…")]`, `#[test(skip)]`, `#[test(skip = "…")]`, or
`#[test(timeout = N)]`, followed by `[async] fn name() { … }` — whose shape is
fixed ([`../testing.md`](../testing.md)):

- the declaration has no parameters, no return type, no generic parameters,
  no `where` clause, and no `pub`;
- an absent group means an ordinary test that must complete normally;
- `should_fail` expects any failure: a panic, a runtime failure, a cancelled
  `async` task, or a timeout;
- `should_panic` expects a panic or runtime failure specifically, not a
  cancelled task, a timeout, or any future non-panic failure;
- `should_panic = "text"` additionally requires the failure message to
  contain `text` as a case-sensitive substring;
- `skip` means the test is not executed; `skip = "reason"` reports a reason;
- `timeout = N` sets a wall-clock deadline in milliseconds; `N` must be an integer
  greater than zero, and an expired deadline is a `timedOut` failure, not a
  panic;
- `should_fail` takes no message; at most one expectation and at most one
  `timeout` may appear; `skip` must be the only option. `should_fail = "text"`,
  duplicate options, mixed expectations, `skip` with any other option, or a
  non-positive/invalid `timeout` are `E0110`;
- the deferred names `category`, `focus`, `retry`, and `threads` are not
  options in this change and are `E0110`;
- the identifier binds no value, and any reference to it is `E0107`.

Written on a `struct`, `enum`, `trait`, or `impl`, `#[test]` is `E0109`
(`` attribute `test` is not valid on a struct ``). When the full test form
does not match, a `#[test]`, `#[test()]`, `#[test(should_fail)]`,
`#[test(should_panic)]`, `#[test(skip)]`, `#[test(timeout = 100)]`, or
`#[test(...)]` on the non-matching `fn` — including a `fn` with parameters, a
return type, generics, a `where` clause, or `pub` — is `E0110`
(`` attribute `test` takes `should_fail`, `should_panic`,
`should_panic = "…"`, `skip`, `skip = "…"`, or `timeout = N` ``). A
`#[test(...)]` group that the attribute grammar cannot parse — a positional
string or numeral, a float, or a boolean, for example — is a syntax error.
When the sequence `#[test]` or `#[test(options)]` followed by `async`? `fn`
Identifier `(` `)` Block appears at file scope, the declaration is a
`TestDecl` and no registry lookup runs ([`../grammar.md`](../grammar.md)).
