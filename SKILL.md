---
name: woodmart-mcp
description: "Use for WoodMart site work through the wood MCP abilities (woodmart/*): finding or creating content, editing Gutenberg blocks or Elementor elements, theme settings and header logos, sliders, template library and WoodMart layouts. This file is the router — pick gutenberg.md or elementor.md for the builder in use before editing content."
---

# WoodMart MCP — skill router

This package ships one skill per page builder:

| File | Use for | Write model |
|------|---------|-------------|
| `gutenberg.md` | Pages built with Gutenberg blocks (`woodmart/gutenberg-*`) | queued; live only after AI Block Queue finalization |
| `elementor.md` | Pages built with Elementor (`woodmart/elementor-*`) | synchronous (`applied`), no queue |

## Choose the skill

1. **Existing content — the target's builder wins:**
   - `woodmart/elementor-get-content` returns a tree (or the target reports `built_with_elementor`) → read `elementor.md`;
   - `woodmart/gutenberg-get-content` returns blocks → read `gutenberg.md`.
   - Gutenberg abilities return empty on an Elementor page without a warning, and vice versa — never edit blind.
2. **New content — WoodMart Theme Settings decide.** `woodmart/create-content`, `woodmart/elementor-create-content` and `woodmart/create-layout` default to `builder: "auto"` (Theme Settings); an explicit `builder` is a deliberate override, and a mismatch fails with `wd_ai_wrong_builder` and a hint.
3. Read **only the chosen file** and follow it; the shared rules below always apply.

## Shared rules (both builders)

- Every write validates its input and returns `errors`/`warnings`; an invalid tree saves nothing.
- WoodMart MCP cannot upload media, set featured images, categories/tags, excerpts, dates or authors, and cannot delete content. Resolve ids with the live queries, never invent ids, and hand the user a "field → value" table for what the abilities cannot do.
