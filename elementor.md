# Working with WoodMart via MCP — Elementor

Use this file when the site builder (WoodMart Theme Settings) is Elementor, or when the target
page/slide/layout is Elementor-built. For Gutenberg pages read `gutenberg.md` instead.

## Use live ability documentation

Everything runs through the wood MCP adapter. Use `mcp-adapter-discover-abilities` for the current
ability list and `mcp-adapter-get-ability-info` before the first use of a write ability, or whenever
its parameters are uncertain. Treat the live description, input schema, response fields, warnings and
`user_instruction` as authoritative; do not rely on remembered parameter lists.

## Routing

| Intent | Start with |
|--------|------------|
| Read an element tree | `woodmart/elementor-get-content` (default `detail: skeleton`; `element_id` for one element) |
| Discover widgets and their settings | `woodmart/elementor-list-widgets`, then `woodmart/elementor-describe-widget` |
| Check a tree without saving | `woodmart/elementor-validate-elements` |
| Change part of a page | `woodmart/elementor-edit-elements` (operations + `expected_content_hash`) |
| Fill an empty target or deliberately rebuild all of it | `woodmart/elementor-replace-content` |
| Create a page/post/portfolio/HTML block | `woodmart/elementor-create-content` |
| Read or change document settings (page, slide `wd_*`) | `woodmart/elementor-get-document-settings` → `woodmart/elementor-update-document-settings` |
| Create or manage sliders | `woodmart/list-sliders`, `woodmart/save-slider` → `woodmart/save-slide` |
| Reuse a demo section | `woodmart/gutenberg-search-templates`, then `woodmart/gutenberg-get-template` with `builder: "elementor"` |
| Create/assign a layout | `woodmart/list-layout-types`, then `woodmart/create-layout`, `woodmart/set-layout-conditions` |

Builder default: `woodmart/create-content`, `woodmart/elementor-create-content` and
`woodmart/create-layout` accept `builder: "auto" | "gutenberg" | "elementor"`. `auto` follows Theme
Settings; an explicit value is a deliberate override. A mismatch without the override fails with
`wd_ai_wrong_builder` and a hint.

## Workflow (writes are synchronous)

There is no block queue for Elementor: every write applies immediately (`applied: true`). Nothing to
finalize.

1. **Read first.** `elementor-get-content` returns the tree with `elementId`s, settings and
   `content_hash`. Use `detail: skeleton` by default, `element_id` for a single element's full
   settings.
2. **Describe before authoring.** `elementor-describe-widget` is an index: it returns content/layout
   controls plus **collapsed groups** (typography, background, icon, …) with an `enable` control and
   `expand: prefix=<group>`; `conditional: true` marks controls that only apply when their condition
   holds. Drill down with `prefix`, `section` or `names` instead of requesting everything. Follow the
   returned `guidance`, examples, units and responsive naming (`<name>_tablet`, `<name>_mobile`).
   Do not invent widget names, control keys or select values.
3. **Validate.** `elementor-validate-elements` is a dry run for a hand-composed tree or for the same
   `operations` — it reports `errors` (unknown widget/elType, bad nesting, duplicate ids, untyped
   atomic values) and `warnings` (unknown settings, invalid options, value shapes). Errors prevent
   the write; warnings tell you what Elementor would store and ignore.
4. **Write tiny.** Never rebuild a target to change part of it: send only the operations and keys that
   change, addressed by `elementId`, with `expected_content_hash` from the latest `get-content`. On a
   conflict, re-read the target and rebuild the operations. `replace-content` is only for an empty
   target or an explicit full rebuild; it refuses content loss unless `allow_content_loss` is set.
   `create-content` validates first and creates nothing if invalid.
5. **Elements without ids get one generated**; omit `id` on new elements and never reuse an existing
   id. Only one write should target a document at a time.
6. **Verify.** Check the page with screenshots at desktop 1440, tablet 820 and mobile 375
   (horizontal overflow, overlapping, empty sections).

## Widget gotchas

- **Unset keys render their defaults.** `wd_title` without `subtitle`/`after_title` shows
  "Subtitle text example" / "Text after title text example". For every text field either set the
  value or explicitly pass `""`.
- `elementor-describe-widget` needs `include_hidden: true` upstream (`elementor-list-widgets`) to see
  layout-only widgets (single product, shop archive, cart, checkout, my account, posts).
- Controls whose value is a shape: sliders `{unit, size}`, dimensions `{top,right,bottom,left,unit}`,
  media `{id, url, size}`, repeater items include `_id`.

## Atomic (v4) elements

- v4 elements (`e-flexbox`, `e-div-block`, `e-grid`, `e-heading`, …) take typed values
  `{"$$type": "…", "value": …}` and keep visual styles in `styles` (local classes), not `settings`.
- Known gap: atomic `styles` are **not validated** — `valid: true` does not guarantee they are kept.
  Verify the front end after writing atomic elements.
- Do not mix v3-only value formats into v4 props; the validator rejects untyped values.

## Document settings

- `elementor-get-document-settings` lists keys (page settings, or a slide's `wd_*` keys) with a
  `settings_hash`; `elementor-update-document-settings` merges only the keys you send (a `null` value
  removes a key) and detects concurrent changes via `expected_settings_hash`. Elementor itself
  replaces the whole settings array on save — the ability merges for you.

## Sliders

A `wd_slider` widget does not contain slides: it references a WoodMart slider (`woodmart_slider`
term; each slide is a `woodmart_slide` post) by **slug**.

1. `woodmart/save-slider` — create/update the slider and its settings (height for all three
   viewports, `stretch_slider`, arrows, pagination, autoplay). Updates merge; `validate_only` checks
   without writing.
2. `woodmart/save-slide` — one call per slide (`title`, `slider`, `order`, `status`, `elements` for
   inner content, `settings` for background/alignment). Slide settings are Elementor document
   settings (`wd_*`); a friendly set (`bg_image`, `image`, `vertical_align`, `content_width`, …) is
   mapped for you. Slide content is a normal Elementor tree — the root must be a container, not a
   bare widget.
3. Place `wd_slider` on the page with `slider = <slug>`.

Design notes: give text on light photos a dark underlay; on mobile add `wd_padding_mobile` (~46px
left/right) and reduce `content_width_mobile` so arrows do not overlap; set `height` for all three
viewports; sliders initialize lazily (verify with `el.swiper.slideTo()`).

## Template library

- The demo catalog is builder-agnostic. `woodmart/gutenberg-get-template` with `builder: "elementor"`
  (or `auto` on an Elementor site) returns an Elementor `elements` tree (ids stripped) plus
  `page_settings`, ready for `elementor-replace-content` on an empty target or for inserts via
  `elementor-edit-elements`.
- This ability does not sideload images: element settings still point at the demo server URLs.
  Localize media separately (import files and replace the URLs) — there is no media ability.

## Layouts

- `woodmart/create-layout` creates a `woodmart_layout`. `predefined_name` imports the predefined
  template in the Theme Settings builder (on Elementor sites the theme imports
  `content-elementor.xml`), conditions are set with `set-layout-conditions`.
- Custom Elementor layout: create it **empty** (no `block_spec`) and fill it with
  `elementor-replace-content` using the returned `target_id`. `block_spec` is Gutenberg-only; on an
  Elementor site it needs the `builder: "gutenberg"` override or the empty-layout path.

## Limits and reporting

- Not available through MCP: media upload, featured images, categories/tags, excerpt/date/author,
  post meta (page-title/footer/sidebar toggles), deleting content. Provide the user a "field →
  value" table for those steps.
- `update-content` changes only metadata (never blocks/elements); publishing and slug/parent changes
  need explicit confirmation and their resulting URL must be reported.
- If every `woodmart/*` ability returns 404 on Elementor 4.3+, the default MCP server was disabled by
  Elementor (option `elementor_mcp_enabled` off). Report it; do not work around it silently.
