# Lexical Structure — Delta: Lowercase Primitive Keywords

> **Delta** — edits the **Keywords** section of `lexical-structure.md`.

The count sentence becomes:

The language has exactly the following closed set of 58 keywords; every other
word is an ordinary identifier unless it is reserved below. No keyword MAY be
used as one.

The table gains a **Primitive types** category, appended after Ownership:

| Category | Keyword | Purpose |
|----------|---------|---------|
| Primitive types | `bool` | The two-valued boolean type |
| | `string` | The immutable UTF-8 text type |
| | `number` | The numeric union type `int \| float` |
| | `int` | 64-bit signed integer type |
| | `float` | 64-bit IEEE-754 binary64 type |
| | `i8` | 8-bit signed integer type |
| | `i16` | 16-bit signed integer type |
| | `i32` | 32-bit signed integer type |
| | `i64` | 64-bit signed integer type |
| | `u8` | 8-bit unsigned integer type |
| | `u16` | 16-bit unsigned integer type |
| | `u32` | 32-bit unsigned integer type |
| | `u64` | 64-bit unsigned integer type |
| | `f32` | 32-bit IEEE-754 binary32 type |
| | `f64` | 64-bit IEEE-754 binary64 type |
| | `null` | The null literal; the type of `null` |
| | `unit` | The unit type |
| | `unknown` | The top type |

(The `null` row replaces the existing `Literals and operators` row for
`null`, so that category keeps six entries and the total is 58.) The
existing notes — `from`/`import` retired, `print` and UI components not
keywords, the retired `test` spelling — are unchanged.
