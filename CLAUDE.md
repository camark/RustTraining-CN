# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains **7 Rust training books** written in Markdown, built using [mdBook](https://rust-lang.github.io/mdBook/). Each book targets developers from different programming backgrounds (C/C++, C#, Python) or focuses on specific topics (Async, Patterns, Type-Driven Correctness, Engineering Practices).

## Architecture

### Repository Structure

```
RustTraining-CN/
├── <book-name>/           # Each book is a self-contained mdBook project
│   ├── book.toml          # mdBook configuration (title, preprocessor, theme)
│   └── src/               # Markdown chapters (ch00-introduction.md, ch01-*.md, etc.)
├── xtask/                 # Cargo workspace member for build/serve/deploy commands
│   └── src/main.rs        # Custom CLI: cargo xtask {build|serve|deploy|clean}
├── site/                  # (generated) Built HTML for local preview
└── docs/                  # (generated) Built HTML for GitHub Pages deployment
```

### Book List

| Directory | Target Audience |
|-----------|-----------------|
| `c-cpp-book/` | C/C++ developers |
| `csharp-book/` | C# developers |
| `python-book/` | Python developers |
| `async-book/` | Async/deep dive |
| `rust-patterns-book/` | Advanced patterns |
| `type-driven-correctness-book/` | Type-level correctness |
| `engineering-book/` | Engineering practices |

### Unified Build System

All books share a common build tooling via `cargo xtask`:

```bash
cargo xtask build      # Build all books to site/
cargo xtask serve      # Build + serve at http://localhost:3000
cargo xtask deploy     # Build all books to docs/ for GitHub Pages
cargo xtask clean      # Remove site/ and docs/
```

To work on a single book:
```bash
cd <book-name> && mdbook serve --open
```

### Translation Convention

All content is translated to Chinese. Files follow naming convention:
- `README_zh_cn.md` — Chinese README variant
- Book content in `<book>/src/ch*.md` is already translated

When adding new files, keep technical terms in English (Rust, `Vec`, `Option`, `trait`, `struct`, etc.) while translating prose to Chinese.

## Key Commands

### Prerequisites
```bash
cargo install mdbook@0.4.52 mdbook-mermaid@0.14.0
```

### Common Operations
```bash
# Build all books
cargo xtask build

# Serve all books with live reload
cargo xtask serve

# Build single book (e.g., for testing one chapter)
cd c-cpp-book && mdbook build

# Clean build artifacts
cargo xtask clean
```

### Serving a Single Book
```bash
cd c-cpp-book && mdbook serve --open  # http://localhost:3000
```

## Content Guidelines

### Chapter Structure

Each chapter follows a consistent pattern:
1. **Opening block** with "What you'll learn" and difficulty indicator (🟢🟡🔴)
2. **Code comparisons** (C/C++|C#|Python vs Rust side-by-side)
3. **Mermaid diagrams** for visual explanations
4. **Exercises** with collapsible `<details>` solutions
5. **Key takeaways** summary

### Code Examples

- Keep Rust code idiomatic; show C/C++/C#/Python equivalents for comparison
- Use `ignore` for non-compilable comparison snippets
- Use `rust,ignore` for Rust snippets that demonstrate concepts but don't compile
- All code comments in translated files should be in Chinese

### Mermaid Diagrams

All books use `mdbook-mermaid` preprocessor. Diagrams are defined inline:
```mermaid
graph TD
    A --> B
```

No external image files needed.
