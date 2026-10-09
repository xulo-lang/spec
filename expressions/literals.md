# Literals

A literal is a source-level expression that denotes a fixed value: a number, a
string, a template, a boolean, `null`, a list, a map, or a tuple.
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
(`let w: U32 = 800`). An integer literal defaults to type `Int`, and a literal
whose value does not fit in its default or coerced type is an error:

```xulo
let big = 99999999999999999999   // error: out of range for Int
let byte: U8 = 300               // error: out of range for U8
```

Outside such a context the type of an integer literal is `Int`, so
`let n = 0xff` infers `Int`. A negative value is not part of the literal: it
is the unary `-` operator applied to a literal, at the `unary` precedence
level (see [`operators.md`](operators.md)). See
[`../types/primitive-types.md`](../types/primitive-types.md).

## Float literals

Float literals contain a decimal point and a fractional part.

```xulo
let pi = 3.14
let half = 0.5
```

A float literal defaults to type `Float`. Exponent notation does not exist:
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
`String`. Neither quote form interpolates — interpolation belongs exclusively
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

Every expression interpolated by `${}` MUST be a base type (`Int`, `Float`,
`Boolean`, `String`) or a type that implements the built-in `ToString`
protocol; otherwise the program is ill-formed at compile time.

```xulo
let count = 42
let ok = true
let good = `count=${count} ok=${ok}`   // OK: Int and Boolean convert

struct User { name: String, age: Int }
let u = User(name: "Alice", age: 30)
let bad = `user: ${u}`                 // error: User has no string form
```

The type of a template literal is always `String`, regardless of its
interpolations. See [`../types/primitive-types.md`](../types/primitive-types.md)
and [`../type-system/checking-rules.md`](../type-system/checking-rules.md).

## Boolean and Null literals

```xulo
let ok = true
let no = false
let missing = null
let maybe: String? = null
```

`true` and `false` have type `Boolean`. `null` has type `Null` and inhabits
every optional type `T?`; assigning `null` to a non-optional type is an error.
See [`../types/primitive-types.md`](../types/primitive-types.md).

## List literals

A list literal is a comma-separated sequence of expressions in brackets.

```xulo
let xs = [1, 2, 3]
let ys = [1, 2, 3,]
let merged = [...xs, 4]
let rows = [[1, 2], [3]]          // List<List<Int>>
```

The element type is the common type of the elements, so `[1, 2, 3]` has type
`List<Int>`. A trailing comma is allowed. The prefix spread `...expr` MAY
appear as an element and its operand MUST be a `List`; the elements of that
list are appended in order. See
[`../types/composite-types.md`](../types/composite-types.md).

An empty list literal `[]` has no elements from which to infer an element
type, so it is well-formed only where the expected type is already known —
from a type annotation, a parameter type, or another contextual type. When no
expected type is available, the literal MUST be annotated:

```xulo
let xs: List<Int> = []        // OK: element type given by the annotation
let ys = []                   // error: no expected type to infer from
```

## Map literals

A map literal is a brace-delimited, comma-separated list of `key: value`
entries and evaluates to a map.

```xulo
let user = { name: "lyy", age: 30 }   // Map<String, String | Int>
let mut updating = { ...user, age: 31 }
```

Keys are identifiers; each becomes the string key of the entry, and the
literal's default type is `Map<String, C>` with `C` the union of the value
types. Keys that are not identifiers — arbitrary string keys, computed keys —
use the typed form `Map<K, V>{ … }`
([`../types/composite-types.md`](../types/composite-types.md)). A literal MAY
be annotated with a map type, including through an alias:

```xulo
type User = Map<String, String | Int>
let u: User = { name: "lyy", age: 30 }
```

A type is written exactly once. When an expected type `Map<K, V>` is in
scope — a `let` annotation, a declared return type, a parameter, an
assignment target, any other checking position, after alias expansion — and
every key of the literal is an identifier (or the literal is empty), the
literal MUST use the brace form: the typed form repeats the type the context
already gives and is a compile-time error (`E0221`). The typed form is written in
exactly two situations: no expected map type is in effect, or at least one key
is not an identifier — which only the typed form can write, and which keeps it
legal next to an annotation.

```xulo
fn counts(): Map<String, Int> {
  return { "a": 1 }                    // the signature already gives the type
}

let bad: Map<String, Int> = Map<String, Int>{ "a": 1 }  // error[E0221]: repeats the annotation
let ages: Map<Int, String> = Map<Int, String>{ 30: "thirty" }  // OK: a key that is not an identifier
```

A trailing comma is allowed, and the prefix spread `...expr` MAY appear as an
entry; its operand MUST be a `Map`, and when the same key occurs more than
once, the later occurrence wins — except that a key written twice directly in
one literal is a compile-time error. An empty `{}` determines its type from
the expected type, exactly like `[]`, and is a compile-time error when nothing
determines it:

```xulo
let empty: Map<String, Int> = {}    // OK: entry type given by the annotation
let lost = {}                       // error: no expected type to infer from
```

At statement position `{` always introduces a block, so an expression statement
MUST NOT begin with `{`. To use a map literal where a leading `{` would be
ambiguous, parenthesize it:

```xulo
{ a: 1 }          // error: `{` starts a block here, not a literal
({ a: 1 })        // OK: a map literal, parenthesized
```

## Tuple literals

A tuple literal is a parenthesized, comma-separated sequence of at least two
expressions:

```xulo
let p = (10, "ten")
let annotated: (Int, String) = (10, "ten")
let more = (1, 2, 3,)         // trailing comma allowed
```

- The comma decides the reading: `( e )` groups a single expression, and
  `( e₁, e₂, … )` builds a tuple. A one-element tuple and an empty tuple
  literal do not exist, and `( )` is not well-formed.
- The type of `(e₁, …, eₙ)` is `(T₁, …, Tₙ)`, the tuple of the element
  types. With an expected tuple type in scope, each element checks against
  the expected element type at that position — an integer literal in a
  `(Float, …)` context becomes a `Float`, exactly as an annotation would
  cause ([`../type-system/checking-rules.md`](../type-system/checking-rules.md)).
- The prefix spread `...expr` does not appear in tuple literals: it exists
  only inside list and map literals ([`operators.md`](operators.md)).
- Element evaluation is left to right, like every other composite literal,
  and a tuple literal in statement position follows the statement-value rule
  ([`../statements/expression-statements.md`](../statements/expression-statements.md)).
- For named rather than positional grouping use a `struct` (named fields) or
  a `Map` (dynamic keys); a tuple is chosen when the positions themselves
  carry the meaning
  ([`../types/composite-types.md`](../types/composite-types.md)).
