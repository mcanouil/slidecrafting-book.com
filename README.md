# slidecrafting.com

Source for [slidecrafting-book.com](https://slidecrafting-book.com) — a guide to making beautiful slides with Reveal.js and Quarto.

## Structure

- `*.qmd` — book chapter source files
- `examples/` — standalone Reveal.js slide decks embedded as iframes in the book
- `_book/` — built output (do not edit directly)

## Building

The site has two components that must be rendered in order:

```bash
# 1. Render the example slide decks
quarto render examples/

# 2. Render the book
quarto render
```

The book copies from `examples/` during its render, so the examples must be built first.

## Adding new examples

Create a new `.qmd` in the appropriate `examples/` subdirectory. Use `format: revealjs` without `self-contained: true`:

```yaml
---
format:
  revealjs: {}
---
```

Or with options:

```yaml
---
format:
  revealjs:
    theme: [default, my-theme.scss]
---
```

Do **not** add `self-contained: true` — examples share a common `libs/` directory to avoid bundling Reveal.js (~11MB) into every file.
