# Documentation Comments (Informative) — Delta: Lowercase Attribute Brackets

> **Delta** — replaces the `#Test` mentions in the test paragraph and the
> adjacency rule of `documentation-comments.md`.

Replace the **Inert code** bullet with:

- **Inert code.** Code in a documentation block is text. Unlike some
  ecosystems, it is **not** executed or type-checked as a test. Tests are
  `#[test]` functions ([`testing.md`](testing.md)).

Replace the attribute-lines bullet with:

- Only whitespace and **attribute lines** (such as `#[test]`) may appear
  between the documentation comment and the declaration. Attributes decorate
  the same declaration and do not break attachment.
