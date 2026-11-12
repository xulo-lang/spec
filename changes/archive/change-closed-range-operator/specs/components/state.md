# State — Delta: Closed Range Operator

> **Delta** — replaces the nested-block example in `components/state.md`.

## Placement

The `Panel` example becomes:

```xulo
fn Panel(): View {
  if true {
    @State let hidden = false // error: nested block
  }

  for i in 0..=3 {
    @State let step = i       // error: nested block
  }

  VStack { }
}
```
