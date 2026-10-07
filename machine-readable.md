---
title: Machine-Readable Grammar Files
description: Formal grammar definitions for parser generators, IDE plugins, and static analysis tools.
---

## Overview

This page provides machine-readable grammar source files (Machine-Readable Grammar) for the XULO language. These files are intended for compiler authors, parser generator tooling, linter/static analyzer developers, and IDE plugin authors to reference and compile directly.

---

## Available Grammar Files

### 1. `xulo.ebnf` (ISO/IEC 14977 Standard)

ISO/IEC 14977 standard EBNF grammar definition. Compatible with most EBNF-based parser generators and format conversion tools.

- **Specification**: [ISO/IEC 14977:1996](https://www.iso.org/standard/26153.html)
- **Encoding**: UTF-8 Plain Text
- **Download**: [Download xulo.ebnf](https://xulo.org/grammars/xulo.ebnf) · [View Raw Source](https://github.com/xulo-lang/xulo/blob/main/crates/xulo-parser/grammar/xulo.ebnf)

### 2. `xulo.pest` (Rust Pest Grammar)

Pest PEG (Parsing Expression Grammar) file for the Rust ecosystem's Pest parser generator.

- **Tool**: [Pest Parser Generator](https://pest.rs/)
- **Download**: [Download xulo.pest](https://xulo.org/grammars/xulo.pest) · [View Raw Source](https://github.com/xulo-lang/xulo/blob/main/crates/xulo-parser/grammar/xulo.pest)

### 3. `xulo-tree-sitter` (Incremental Parser)

Tree-sitter incremental parsing library for modern editors (VS Code, Neovim, Helix, Zed) syntax highlighting and AST navigation.

- **Repository**: [github.com/xulo-lang/xulo-tree-sitter](https://github.com/xulo-lang/xulo-tree-sitter)
- **Supported platforms**: C, Rust, WASM, Node.js

---

## Quick Integration Guide

### Using the Pest Parser in Rust

Add dependencies to `Cargo.toml`:

```toml
[dependencies]
pest = "2.7"
pest_derive = "2.7"
```

Import the grammar in Rust code:

```rust
use pest_derive::Parser;

#[derive(Parser)]
#[grammar = "xulo.pest"] // Path to xulo.pest in your project
pub struct XuloParser;
```

### Integrating Tree-sitter in JavaScript / Node.js

```bash
npm install tree-sitter xulo-tree-sitter
```

```javascript
const Parser = require('tree-sitter');
const Xulo = require('xulo-tree-sitter');

const parser = new Parser();
parser.setLanguage(Xulo);

const tree = parser.parse('fn main() { return 0; }');
console.log(tree.rootNode.toString());
```

### Validating EBNF Grammar

```bash
# Validate rules using ebnf-validator
ebnf-validate ./xulo.ebnf
```

---

## Single Source of Truth

All grammar files listed on this page are automatically generated and synced from the **XULO compiler main repository** during CI/CD builds, ensuring 100% semantic consistency with the latest compiler engine.

- **Main compiler repository**: [`github.com/xulo-lang/xulo`](https://github.com/xulo-lang/xulo)
- **Grammar maintenance directory**: `crates/xulo-parser/grammar/`
