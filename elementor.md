# Working with WoodMart via MCP — Elementor

Use this file when the WoodMart Theme Settings builder is Elementor, or the target page/slide/layout
is Elementor-built. For Gutenberg pages read `gutenberg.md` instead.

Use the live ability docs (`mcp-adapter-discover-abilities`, `mcp-adapter-get-ability-info`) before
the first use of a write ability; the live schema, warnings and `user_instruction` are authoritative.

## Routing

| Intent | Start with |
|--------|------------|
| Read an element tree | `woodmart/elementor-get-content` (`detail: skeleton`; `element_id` for one element) |
| Discover elements and their controls/props | `woodmart/elementor-list-elements` → `woodmart/elementor-describe-element` |
| Change part of a page | `woodmart/elementor-edit-elements` (operations + `expected_content_hash`) |
| Fill an empty target or deliberately rebuild all of it | `woodmart/elementor-replace-content` |
| Create a page/post/portfolio/HTML block | `woodmart/elementor-create-content` |
| Read or change document settings (page, slide `wd_*`) | `woodmart/elementor-get-document-settings` → `woodmart/elementor-update-document-settings` |
| Create or manage sliders | `woodmart/list-sliders`, `woodmart/save-slider` → `woodmart/save-slide` |
| Reuse a demo template | `woodmart/elementor-search-templates` → `woodmart/elementor-get-template` (or pass its `template_id` to a write) |
| Create or assign a layout | `woodmart/list-layout-types` → `woodmart/create-layout`, `woodmart/set-layout-conditions` |

## Workflow (writes are synchronous)

No block queue: every write validates the tree first and applies immediately (`applied: true`);
nothing is saved when it is invalid. Pass `validate_only` to a write to check it without saving.

1. **Read first** with `elementor-get-content` (`detail: skeleton`; `element_id` for one element's full settings).
2. **Describe before authoring.** `elementor-describe-element` indexes one element type: classic (v3) controls or atomic (v4) typed/style props. Drill in with `tab`/`section`/`names`/`prefix`; groups are collapsed unless `collapse: false`. Never invent element names or control keys.
3. **Write tiny.** Only the operations/keys that change, by `elementId`, with `expected_content_hash`. `replace-content` is for an empty target or an explicit rebuild only and needs `allow_content_loss` to drop most of the page; `create-content` validates first. Omit `id` on new elements; never reuse an existing id.
4. **Verify** the front end at desktop 1440 / tablet 820 / mobile 375.

## Gotchas

- Unset text keys render defaults (`wd_title` → "Subtitle text example"); set the value or pass `""`.
- Pass `include_hidden: true` to `elementor-list-elements` to see layout-only widgets (single product, cart, checkout, my account, posts).
- Shape values: sliders `{unit, size}`, dimensions `{top,right,bottom,left,unit}`, media `{id, url, size}`, repeater items include `_id`.
- **Atomic (v4)** elements (`e-flexbox`, `e-grid`, `e-heading`, …) take typed values `{"$$type": "…", "value": …}`; styles live in `styles` (local classes), not `settings`. Never mix v3 formats into v4 props.
- **Slider**: `wd_slider` references a WoodMart slider by slug; slide content is a normal tree whose root must be a container.
- **Layout**: create custom Elementor layouts empty and fill them with `elementor-replace-content` (`block_spec` is Gutenberg-only).
- Demo template images are copied to the media library on save; any left are listed in `images.remaining` (copied on the next save).

An MCP server disabled on Elementor 4.3+ (every `woodmart/*` 404, option `elementor_mcp_enabled` off) must be reported, not worked around.
