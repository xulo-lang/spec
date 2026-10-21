# Functions — Delta: Cross-Reference for the `extern` Forms

> **Delta** — adds one bullet to the **Definition** section of `functions.md`.
> The **Visibility** section is unchanged: `pub` remains the only mechanism
> granting importer visibility, and `pub extern "C" fn …` carries `pub` like
> any other exported function ([`ffi.md`](ffi.md)).

After the closing bullet of **Definition** — “`pub` on a `fn` marks it as
exported from its module; see [Visibility](#visibility).” — one bullet joins:

- The `extern` forms declare the same function shape under a foreign ABI: an
  import block `extern "C" { fn …; }` declares signatures with no body, and a
  `pub extern "C" fn …` exports a body under that ABI. Signature types,
  call rules, and the boundary guarantee are in [`ffi.md`](ffi.md). Neither
  extern form appears in a function body: `Statement` admits a nested `fn`
  and no extern form.
