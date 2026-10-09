# Layout

Components build a tree of `View` values; nothing in that tree has a size or a
position yet. This chapter specifies the contract between the tree and the
pixels: how a node is measured and placed, what the size, spacing, and style
attributes mean, how an interaction finds the node it lands on, and in what
order the whole pipeline runs. Which components exist and which attributes each
of them accepts is library contract ([`view-syntax.md`](view-syntax.md)); the
rules below are the language-level layout contract that programs may rely on,
independent of any particular rendering back end.

## The layout model

A `View` tree is **measured and positioned before painting**. Layout is
constraint-driven:

- Each node receives constraints from its parent — the space the parent offers
  it along each axis — and, under those constraints, proposes a size.
- The parent then assigns the node a position, and the node lays out its own
  children in the same way, recursively, until every node in the tree has a size
  and a position.
- Painting happens only after the whole tree has been laid out, and hit testing
  uses the boxes layout produced (see *Hit testing* below).
- These are statements about what a program may rely on — containment, order,
  and the meanings of the attributes below — not about how a renderer computes
  them.

## Containers and children

A container arranges its children according to its kind:

- **Stack-style containers** arrange their children along a **main axis**, in
  **document order**: `VStack` stacks vertically, so its main axis is vertical;
  `HStack` stacks horizontally, so its main axis is horizontal. The axis across
  which children are placed is the **cross axis**.
- **Containment:** each child is laid out within its parent's bounds. A child's
  position is expressed inside its parent's box, and the parent's box is what
  the parent's parent positions — no child escapes the bounds of its container.
- **Document order is paint order.** A child that comes later in the block is
  painted over a child that comes earlier, and an overlaid container paints its
  children in the same order. Paint order is what makes *topmost* well defined
  for hit testing.

Children contribute to their container in the order they appear in the block,
including the children contributed by `if`, by `for`, and by a flattened
`List<View>` ([`view-syntax.md`](view-syntax.md)).

## Size properties

`width` and `height` set a node's size along the horizontal and vertical axes.
Each accepts a number, or one of the size literals:

| Form | Example | Meaning |
|------|---------|---------|
| Number | `width: 200` | An exact size, in logical pixels |
| `"auto"` | `height: "auto"` | The content's size: the node is sized by what it contains |
| `"fill"` | `height: "fill"` | Take the remaining space offered along that axis |
| Percentage | `width: "100%"` | That share of the parent's size along that axis |

- **Numeric sizes are logical pixels**: device-independent units, not device
  pixels, so the same source lays out the same way on every presentation
  target.
- **`"auto"`** means *content size*: a text node sizes to its text, a container
  sizes to the children it requires together with its own spacing and padding.
- **`"fill"`** means *take what is left*: after siblings with fixed sizes and
  siblings sized `"auto"` have been placed, the space remaining along that axis
  belongs to the `"fill"` nodes. `"fill"` on a node is the flexible size — the
  equivalent of growing to share the leftover space (flex `1`).
- **Leftover space is split equally among `"fill"` siblings.** When more than
  one child of the same container is `"fill"` along the container's main axis,
  the space remaining after the other children are placed is divided in equal
  shares among them; a single `"fill"` child takes all of it.
- `width: "fill"` or `height: "fill"` on a container sets the container's own
  size along that axis from the space its parent offers; `"auto"` on a
  container's cross axis lets the container take the size its children need
  across that axis.

```xulo
VStack(height: "fill") {
  Text("Header")
  VStack(height: "fill") {      // the only "fill" child: takes all that is left
    Text("Body")
  }
  Text("Footer")
}

HStack {
  VStack(width: "fill") { Text("Left") }   // equal shares: 50 / 50
  VStack(width: "fill") { Text("Right") }
}
```

## Spacing and alignment

Five attributes control how children sit relative to each other and to their
container:

| Attribute | Applies to | Type | Meaning |
|-----------|-----------|------|---------|
| `padding` | any node | `Number` | Inner spacing: room between the node's own edge and its content |
| `margin` | any node | `Number` | Outer spacing: room outside the node, between it and its neighbors |
| `spacing` | stack containers | `Number` | Gap between adjacent children along the main axis |
| `alignment` | stack containers | `String` | Cross-axis placement of children |
| `justify` | stack containers | `String` | Main-axis distribution of children |

The values of `alignment` and `justify` are strings, and the values defined
here are `start`, `center`, `end`, and `space-between`:

| Value | `alignment` (cross axis) | `justify` (main axis) |
|-------|--------------------------|------------------------|
| `start` | children toward the beginning of the cross axis | children packed at the beginning |
| `center` | children centered across the cross axis | children centered along the main axis |
| `end` | children toward the end of the cross axis | children packed at the end |
| `space-between` | — | equal gaps between adjacent children, none at the ends |

`spacing` and the gaps implied by `justify` are both spacing, but they answer
different questions: `spacing` fixes the gap between neighbors, while `justify`
distributes the space that is left over after the children have their sizes.

```xulo
VStack(spacing: 8, padding: 4, alignment: "center") {
  Text("Title")
  Text("Subtitle")
}

HStack(justify: "space-between", padding: 4) {
  Text("Left")
  Text("Center")
  Text("Right")
}
```

## Style properties

Style attributes are ordinary named arguments, and these are the documented
ones:

| Attribute | Type | Meaning |
|-----------|------|---------|
| `color` | `String` | Text color |
| `bg` | `String` | Background **color** |
| `bgImage` | `String` | Background **image URL** |
| `border` | `String` | Border color |
| `size` | `Number` | Text size, in logical pixels |
| `weight` | `String` | Text weight, e.g. `"bold"` |
| `radius` | `Number` | Corner radius |
| `opacity` | `Number` | Opacity, from `0.0` to `1.0` |
| `alignment` | `String` | Cross-axis alignment of children |
| `justify` | `String` | Main-axis distribution of children |
| `width`, `height` | `Number` or `String` | Size, as specified in *Size properties* |
| `padding` | `Number` | Inner spacing |
| `margin` | `Number` | Outer spacing |

- **`bg` and `bgImage` are strictly separate.** `bg` accepts colors only and
  `bgImage` accepts image URLs only; an image URL written as `bg`, or a color
  written as `bgImage`, is a compile-time error when the argument is a string
  literal, and the two are never interchangeable.
- `size`, `weight`, and `color` apply to text; `bg`, `border`, `radius`, and
  `opacity` apply to the node itself.
- **Language vs library.** That an attribute is written as a named argument in
  `( )`, how it is matched to a declared parameter, and how its value is
  checked against the parameter's type are language rules
  ([`../expressions/calls.md`](../expressions/calls.md)). Which attribute names
  exist, what type each has, and what each one does is the component library's
  contract; the table above records the documented style vocabulary, and a
  component that does not declare an attribute cannot receive it.

## Hit testing

Interaction regions derive from the boxes layout produced:

- An input event carries a point. The node under that point is found by testing
  the laid-out bounds of the tree: a node is a candidate when the point lies
  within the box layout assigned to it, and its children are then tested in
  turn.
- A click resolves to the **topmost interactive node** containing the point —
  the node painted last among the candidates that handle the interaction. Paint
  order therefore decides ties: a later sibling is above an earlier one
  (*Containers and children*).
- Once the node is chosen, its handler runs
  ([`binding.md`](binding.md)) — synchronously, in the scope of the component
  that wrote the closure, and free to assign `@State`.
- A node that contains the point but handles no interaction **lets the
  interaction fall through to the node beneath it** — the node painted below it
  at that point. Falling through repeats until a node handles the interaction or
  there is nothing beneath it, in which case the event is unused.
- Hit testing observes the boxes layout only: the language defines no separate
  region attributes, so what is clickable is what is visible after layout.

## Text measurement

A text node's size derives from its content and its text attributes: the
characters it holds, its `size`, and its `weight` determine how much room it
takes, and layout treats that room as the node's content size — an `"auto"`
dimension takes it, and a container measures it like any other child. No other
text measurement property is part of this specification.

## Rendering pipeline (normative outline)

One pass of a rendered program runs in this order:

1. **Build.** The entry point `main` is invoked, and component bodies run,
   producing the `View` tree for the pass; children are collected from
   arguments, blocks, `if`, `for`, and flattened lists
   ([`view-syntax.md`](view-syntax.md)).
2. **Layout.** The tree is measured and positioned under the rules of this
   chapter: constraints flow down, sizes and positions flow back up.
3. **Paint and hit-test.** The tree is painted in document order, and the
   interaction regions are the boxes layout produced.
4. **Interaction.** An input event resolves to the topmost interactive node
   containing the point, and its handler runs synchronously
   ([`binding.md`](binding.md)).
5. **Re-run.** An assignment to `@State`, or a store update, schedules a
   re-run: the owning component's body runs again and produces a fresh tree for
   that component ([`state.md`](state.md)), and the pipeline continues from
   layout for the new tree.
6. **Effects.** After each run completes and its tree is committed, the
   component's due effects are invoked in declaration order, with their
   cleanups first ([`effect.md`](effect.md)).

Steps 1 through 3 describe every pass; steps 4 through 6 describe what happens
while the program is live. A program whose `main` does not declare `View` stops
before step 1: it runs headlessly and never lays out or paints anything
([`view-syntax.md`](view-syntax.md)).
