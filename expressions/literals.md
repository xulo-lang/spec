# Literals

A literal is a source-level expression that denotes a fixed value: a number, a
string, a template, a boolean, `null`, a list, an object, or a tuple.
Literals are value expressions with no side effects. Ranges are not literals —
they are built from the range operators and are specified in
[`operators.md`](operators.md).

## Integer literals

Integer literals are written in decimal (`42`), hexadecimal (`0xff`), binary
(`0b1010`), or octal (`0o77`) form.

```xulo
let a = 42
let mask = 0xff
let flags = 0b1010
let perms = 0o77
```

Digit separators and type suffixes do not exist: `1_000` and `42u32` are not
well-formed, and a fixed-bit type is obtained by contextual coercion instead
(`let w: u32 = 800`). An integer literal defaults to type `int`, and a literal
whose value does not fit in its default or coerced type is an error:

```xulo
let big = 99999999999999999999   // error: out of range for int
let byte: u8 = 300               // error: out of range for u8
```

Outside such a context the type of an integer literal is `int`, so
`let n = 0xff` infers `int`. A negative value is not part of the literal: it
is the unary `-` operator applied to a literal, at the `unary` precedence
level (see [`operators.md`](operators.md)). See
[`../types/primitive-types.md`](../types/primitive-types.md).

## Float literals

Float literals contain a decimal point and a fractional part.

```xulo
let pi = 3.14
let half = 0.5
```

A float literal defaults to type `float`. Exponent notation does not exist:
`1e3` is not well-formed. A trailing decimal point without a fractional part
is not well-formed either — `1.` is invalid, while `0.5` is valid. See
[`../types/primitive-types.md`](../types/primitive-types.md).

## String literals

String literals are delimited by double quotes or single quotes and are
interchangeable.

```xulo
let a = "hello"
let b = 'hello'
let tab = "col1\tcol2"
let uni = "\u00e9 \u{1F44B}"
```

The escape sequences are `\"`, `\'`, `\\`, `\n`, `\t`, `\r`, `\uXXXX`, and
`\u{...}`. A string literal MAY NOT contain a raw newline; to write a
multi-line string, use a template literal. The type of a string literal is
`string`. Neither quote form interpolates — interpolation belongs exclusively
to templates. See [`../types/primitive-types.md`](../types/primitive-types.md).

## Template literals

A template literal is delimited by backticks, MAY span lines, and interpolates
`${expr}` placeholders.

```xulo
let name = "Xulo"
let n = 3
print(`Hello, ${name}!`)
print(`${n}.${n} = ${n * n}`)

let banner = `line one
line two`
```

Inside a template, `` \` `` escapes a backtick and `\$` escapes a dollar sign,
so `\${not interp}` is literal text, and every escape sequence of a string
literal is also accepted in template text. Templates nest: the expression
inside `${}` is an ordinary expression evaluated in the enclosing scope under
all normal rules, and it MAY itself contain a template — for example
`` `${`inner: ${n * 2}`}` ``.

Every expression interpolated by `${}` MUST be a base type (`int`, `float`,
`boolean`, `string`) or a type that implements the built-in `ToString`
protocol; otherwise the program is ill-formed at compile time.

```xulo
let count = 42
let ok = true
let good = `count=${count} ok=${ok}`   // OK: int and boolean convert

struct User { name: string, age: int }
let u = User(name: "Alice", age: 30)
let bad = `user: ${u}`                 // error: User has no string form
```

The type of a template literal is always `string`, regardless of its
interpolations. See [`../types/primitive-types.md`](../types/primitive-types.md)
and [`../type-system/checking-rules.md`](../type-system/checking-rules.md).

## Boolean and null literals

```xulo
let ok = true
let no = false
let missing = null
let maybe: string? = null
```

`true` and `false` have type `boolean`. `null` has type `null` and inhabits
every optional type `T?`; assigning `null` to a non-optional type is an error.
See [`../types/primitive-types.md`](../types/primitive-types.md).

## List literals

A list literal is a comma-separated sequence of expressions in brackets.

```xulo
let xs = [1, 2, 3]
let ys = [1, 2, 3,]
let merged = [...xs, 4]
let rows = [[1, 2], [3]]          // list<list<int>>
```

The element type is the common type of the elements, so `[1, 2, 3]` has type
`list<int>`. A trailing comma is allowed. The prefix spread `...expr` MAY
appear as an element and its operand MUST be a `list`; the elements of that
list are appended in order. See
[`../types/composite-types.md`](../types/composite-types.md).

An empty list literal `[]` has no elements from which to infer an element
type, so it is well-formed only where the expected type is already known —
from a type annotation, a parameter type, or another contextual type. When no
expected type is available, the literal MUST be annotated:

```xulo
let xs: list<int> = []        // OK: element type given by the annotation
let ys = []                   // error: no expected type to infer from
```

## Object literals

An object literal is a brace-delimited, comma-separated list of `key: value`
fields and evaluates to an anonymous structural object.

```xulo
let user = { name: "lyy", age: 30 }
let empty = {}
let updated = { ...user, age: 31 }
```

Keys are identifiers. Object literals do not accept string keys; a value with
arbitrary string keys is a `map`, which is produced by the built-in map
facilities rather than by a literal (see
[`../types/composite-types.md`](../types/composite-types.md)). A named shape is
expressed with a `type` alias, and an object literal MAY be annotated with that
alias:

```xulo
type User = { name: string, age: int }
let u: User = { name: "lyy", age: 30 }
```

A trailing comma is allowed, `{}` is the empty object, and the prefix spread
`...expr` MAY appear as a field; its operand MUST be an object, and when the
same key occurs more than once, the later occurrence wins.

At statement position `{` always introduces a block, so an expression statement
MUST NOT begin with `{`. To use an object literal where a leading `{` would be
ambiguous, parenthesize it:

```xulo
{ a: 1 }          // error: `{` starts a block here, not a literal
({ a: 1 })        // OK: an object literal, parenthesized
```

## Tuple literals

A tuple literal is a parenthesized, comma-separated sequence of at least two
expressions:

```xulo
let p = (10, "ten")
let annotated: (int, string) = (10, "ten")
let more = (1, 2, 3,)         // trailing comma allowed
```

- The comma decides the reading: `( e )` groups a single expression, and
  `( e₁, e₂, … )` builds a tuple. A one-element tuple and an empty tuple
  literal do not exist, and `( )` is not well-formed.
- The type of `(e₁, …, eₙ)` is `(T₁, …, Tₙ)`, the tuple of the element
  types. With an expected tuple type in scope, each element checks against
  the expected element type at that position — an integer literal in a
  `(float, …)` context becomes a `float`, exactly as an annotation would
  cause ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
- The prefix spread `...expr` does not appear in tuple literals: it exists
  only inside list and object literals ([`operators.md`](operators.md)).
- Element evaluation is left to right, like every other composite literal,
  and a tuple literal in statement position follows the statement-value rule
  ([`../statements/expression-statements.md`](../statements/expression-statements.md)).
- For named rather than positional grouping use a `struct` or an `object`;
  a tuple is chosen when the positions themselves carry the meaning
  ([`../types/composite-types.md`](../types/composite-types.md)).
