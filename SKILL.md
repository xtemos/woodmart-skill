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
2. **New content — WoodMart Theme Settings are the default.** The page builder selected there is what `woodmart/create-content`, `woodmart/elementor-create-content` and `woodmart/create-layout` use by default (`builder: "auto"`). If the user explicitly asks for the other builder ("зроби на Gutenberg", "make it Elementor"), pass `builder: "gutenberg"` / `builder: "elementor"` in that call and follow that builder's skill. A mismatch without the override fails with `wd_ai_wrong_builder` and a hint.
3. Read **only the chosen file** and follow it; the shared rules below always apply.

## Shared rules (both builders)

- Use live documentation: `mcp-adapter-discover-abilities` for the ability list and `mcp-adapter-get-ability-info` before the first use of a write ability or when parameters are uncertain. The live schema, warnings and `user_instruction` are authoritative.
- Never rebuild a target to change part of it — edit only what must change, addressed by the builder's stable ids.
- Validate before writing (the write's own validation, `elementor-validate-elements`, or the Gutenberg normalize path) and read `errors` and `warnings`. Elementor silently drops unknown widgets; Gutenberg queues fail as a whole.
- Publishing and URL changes (slug, parent) are outward-facing: only on an explicit request or confirmation, and report the actual result.
- WoodMart MCP cannot upload media, set featured images, categories/tags, excerpts, dates or authors, and cannot delete content. Resolve ids with the live queries and hand the user a "field → value" table for what the abilities cannot do. Never invent ids.
- After writing a page, verify desktop 1440 / tablet 820 / mobile 375 (screenshots, horizontal overflow, overlaps).
