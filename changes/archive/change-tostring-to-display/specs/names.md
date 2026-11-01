# Names — Delta: String Form — Display, Debug, and Format

> **Delta** — replaces the trait-naming example of the conventions paragraph
> of `names.md`. `Display` becomes a reserved prelude protocol and can no
> longer exemplify a name a program chooses.

The paragraph becomes:

Traits are named for the capability they describe, without an `I` prefix:
`Area`, not `IArea`; `Clone`, not `IClone`. Component names follow
UpperCamelCase, which is how a reader recognizes a component at a use site;
concrete component names such as `Text` and `Button` are ordinary imported
names, not reserved words. Names SHOULD be descriptive rather than abbreviated,
and SHOULD NOT repeat their kind in the name (`userNames` for a list of names,
not `userNamesList`).
