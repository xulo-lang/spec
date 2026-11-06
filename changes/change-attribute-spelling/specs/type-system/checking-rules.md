# Checking Rules — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the test-identifier bullet in
> `type-system/checking-rules.md`.

Replace the bullet with:

- The identifier of a `#[test]` function binds no value; any reference to it —
  a call, a value use, or a path — is `E0107`
  ([`../testing.md`](../testing.md)).
