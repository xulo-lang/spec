# Testing — Delta: Map Lookup Example

> **Delta** — replaces the absent-key test example in `testing.md`; the
> surrounding examples are unchanged.

```xulo
#Test
fn absentKeyIsNull() {
  let m = { a: 1 }
  assert(m.get("z") == null, "absent key is null")
}
```
