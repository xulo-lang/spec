# Introduction — Delta: Closed Range Operator

> **Delta** — replaces the `sum_to` example and its range bullet in
> `introduction.md`.

## A Guided Example

The `sum_to` function becomes:

```xulo
// `let mut` makes a binding assignable; `let` bindings are not.
fn sum_to(n: int): int {
  let mut acc = 0
  for i in 1..=n {
    acc = acc + i
  }
  acc
}
```

The bullet that explains it becomes:

- `let mut acc = 0` creates a mutable binding — `acc = acc + i` would be an
  error on a plain `let`. `1..=n` is a closed range, and `for i in 1..=n`
  iterates it.
