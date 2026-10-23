---
name: woodmart-mcp
description: "Use for any work on a WoodMart site through the woodmart MCP abilities (woodmart/*): pages and their Gutenberg blocks or Elementor elements, theme settings and presets, headers and logos, menus and mega menus, products, attributes and swatches, categories, media, popups, floating blocks, labels, size guides, display conditions, layouts, sliders, sidebars and widgets, checkout fields."
---

# Working with WoodMart via MCP abilities

Everything runs through the MCP adapter: `mcp-adapter-discover-abilities` lists the abilities,
`mcp-adapter-get-ability-info` returns one ability's input schema, `mcp-adapter-execute-ability` runs it.
Read `get-ability-info` before the first call of each ability. The live description, schema, `warnings`,
`hint` and `user_instruction` are authoritative; do not rely on remembered parameter names.

## Where to start

| Task | Start with |
|------|------------|
| Find or list content of any type | `search-content` (`query`, `post_types`); each result has `builder` |
| Title, slug, status, parent, excerpt, featured image, date, author | `update-content` |
| Undo a content change, see older versions | `list-revisions` → `get-revision` (diff with now) → `restore-revision`; undo the restore with its `undo_revision_id` |
| See a draft on the front end (browser tool, no login) | `create-preview-link` (page, post, product, portfolio; HTML block, layout, popup, floating block as in the editor preview) |
| Page/post/product/term options (page title, sidebar, header off, product layout…), popup and rule style fields | `get-metaboxes` → `update-metaboxes` |
| Theme setting for the whole site | `list-theme-settings` → `get-theme-setting` → `update-theme-setting(s)` |
| Theme setting only on some pages | preset: `list-theme-settings-presets` → `save-theme-settings-preset` → `update-theme-settings` with `preset_id` |
| Back up Theme Settings before many changes; undo them | `create-theme-settings-backup`; later `list-theme-settings-backups` → `compare-theme-settings-backup` (what differs now) → `update-theme-settings` with the `backup_value` of the settings to restore |
| Site title, tagline, language, home page and posts page; Settings → Media image sizes (then `regenerate-media`) | `get-site-settings` → `update-site-settings` |
| Appearance → Customize (WooCommerce image sizes, store notice, catalog, additional CSS…) | `get-customizer-settings` → `update-customizer-settings` |
| Header, logo, header elements (desktop and mobile columns hold separate elements, e.g. two logos) | `list-headers` → `get-header` → `describe-header-element` → `edit-header` |
| New header; where a header shows (site default, post types, single pages) | `create-header`; `set-header-assignment` |
| Menu items, mega menu (also the menu a header shows) | `list-menus` → `get-menu` → `describe-menu-item-settings` → `edit-menu`; `create-menu` |
| Media | `list-media` first (search by file name, URL or ID); `upload-media` (a URL) or `request-media-upload` (local files); `update-media` for alt/title; `edit-image` to crop, rotate, flip, scale or restore the original; `regenerate-media` after the image sizes change |
| Categories, tags, attributes, swatches (a value becomes a swatch through its term's `swatch_text` / `swatch_color` / `swatch_image_id`) | `list-taxonomies` → `list-terms` → `create-term` / `update-term` / `set-object-terms`; `create-product-attribute` / `update-product-attribute` (name, slug, swatch style) |
| Products (a variable product's prices live in its variations; a grouped product's price comes from its `children`) | `create-simple-product` / `create-variable-product` / `create-external-product` (affiliate: `product_url`, `button_text`) / `create-grouped-product` (`children`); `get-product` → `update-product` (WooCommerce fields); `get-product-woodmart-fields` → `update-product-woodmart-fields` (WoodMart fields: size guide, 360, videos, bundles, variation images); `update-product-variations` (prices, stock, new variations) |
| Product reviews (read only) | `get-product-reviews` |
| Coupons | `get-coupons` → `create-coupon` / `update-coupon` |
| WooCommerce settings (currency, store address, pages, accounts, emails) | `get-store-settings` → `update-store-settings` |
| Shipping zones and methods; taxes; payment methods | `get-shipping` → `update-shipping`; `get-taxes` → `update-taxes`; `get-payment-gateways` → `update-payment-gateways` |
| Store live or "coming soon" | `get-store-visibility` → `update-store-visibility` |
| Where and when (triggers) a popup, floating block, label, size guide, product tab or Woo rule shows | `get-display-conditions` → `set-display-conditions` |
| Woo rules (discounts, free gifts, delivery, bundles, linked variations) | `create-woo-rule`, then conditions and `update-metaboxes` |
| Layouts (shop, product, cart, blog…) | `list-layout-types` → `create-layout` → `set-layout-conditions` |
| Sliders | `list-sliders` → `save-slider` → `save-slide` |
| Sidebars, widgets | `list-sidebars` → `describe-widget` → `edit-sidebar-widgets`; `create-sidebar` |
| Checkout fields | `get-checkout-fields` → `update-checkout-fields` |
| Page content (blocks / elements) | the builder section below |

## Rules for every task

1. **Resolve, never invent.** Every id (post, term, attachment, menu, header element, widget) comes from a
   live list/search/get call. If something the user named does not exist (a category, a menu, a size guide),
   say so and offer to create it; do not silently substitute a lookalike. A stored id can point to a deleted
   file: `list-media` with `ids` reports `missing_ids`.
2. **Reuse before building.** Look for what the site already has (a similar section, HTML block, menu, image
   in `list-media`) and for a WoodMart demo template (`*-search-templates`) before authoring from scratch.
   A close template with other texts is still the better start: insert it and change only the texts (its demo
   images are copied to the media library). Prefer WoodMart elements (`wd/*` blocks, `wd_*` widgets) over core
   ones when both fit.
3. **Read, then write only the difference.** Read the current state right before a write and send only what
   changes, with the hash that read returned. Do not send values equal to the defaults. Where a description
   says a list is complete or replaces the old one (display and layout conditions, `set-object-terms`
   replace, product `category_ids`), send every entry that must remain.
4. **Check before saving.** When an ability has `validate_only`, run it first, then the same call without it.
   Fix every reported error; never resend an unchanged payload. Abilities without `validate_only` (e.g.
   `create-term`, `upload-media`) apply at once and cannot be undone through MCP: finish every lookup and get
   the user's answers before the first of them.
5. **Ask before outward-facing changes** unless the user asked for exactly that: publishing; changing the
   slug, parent or date of published content (moves its URL); the default header; assigning a menu location;
   a base theme setting that changes the whole site. Create new things as drafts, finish content, settings
   and conditions, then publish: popups, floating blocks and labels without conditions show everywhere.
   (A mega menu accepts only a published HTML block.)
6. **No deletion.** The woodmart abilities cannot delete content, terms, menus, headers, sliders or media
   (only menu items and widgets can be removed, with `edit-menu` / `edit-sidebar-widgets`). Tell the user what
   to remove in wp-admin; never imitate deletion by blanking content.
7. **Report what really happened.** Relay `warnings` (a moved URL, a preset override, a setting with no
   effect, an empty menu location). List what you could not do as "where → field → value" for the user.
   Re-read after writing. Check visible changes on the front end at desktop, tablet and mobile widths when you
   can open the site; otherwise ask the user to check them.

If the woodmart MCP is unavailable or every call fails, stop and tell the user the server name and error.
Do not continue through wp-cli, PHP, SQL or hand-built REST calls.

## Design: match the site

1. **Take values from the site, not from taste.** Before authoring a section, read a similar section of the
   site and reuse its spacing, heading sizes, colors and radius; when the site has none, use the closest
   WoodMart template's values.
2. **Inherit globals.** Fonts, colors and rounding come from Theme Settings. Use theme color variables
   (`var(--wd-primary-color)`, or the block's color-variable attribute) instead of hex copies; set a value
   only where it differs from the global one.
3. **One spacing system.** Space sections with the bottom margin of the upper one; padding only on blocks
   with a background or border; space items with the container gap.
4. **Theme gap.** The theme leaves 40px between the content and both the header and the footer. For a page
   that must touch the header, set `margin-top: -40` on its first block or container; to touch the footer,
   `margin-bottom: -40` on the last one. Otherwise leave the gap alone.
5. **No wrapper without a reason** (a background, a column, sticky, its own responsive behavior).
6. **Responsive.** Reduce large headings and spacing on tablet and mobile; 4+ columns become 2 on tablet and
   1 on mobile (or a carousel when there are many items). A fixed px width needs `100%` on tablet. No `calc()`,
   `min()` or `clamp()` in width fields, no `<br>` to break lines.
7. **Say what you invented.** Texts, prices, policies or images the user did not give are placeholders:
   list them in the report. Ask for missing images instead of uploading random ones.
8. **Wishlist and Compare.** If the site needs them and the pages do not exist, create them with
   `gutenberg-create-content` or `elementor-create-content` (by the site's builder); then pick them in Theme
   Settings (`wishlist_page`, `compare_page`).

## Page content: choose the builder

- **Existing content:** the `builder` from `search-content` decides. Gutenberg abilities read an Elementor
  page as empty and vice versa, so never edit blind.
- **New content:** Theme Settings decide (`get-theme-setting` `current_builder`: `native` = Gutenberg,
  `external` = the active builder plugin). A wrong-builder write fails with `wd_ai_wrong_builder` and a hint;
  pass an explicit `builder` only when the user asked for that builder.
- **Custom labels and size guides** never use Elementor: their content is Gutenberg blocks on every site (use the
  Gutenberg section); a size guide's table is the `woodmart_sguide` field in `update-metaboxes`.
- There are no WPBakery abilities: never edit a WPBakery page with the Gutenberg or Elementor ones; tell the user.

## Gutenberg

| Intent | Start with |
|--------|------------|
| Create a page, post, portfolio project, HTML block, popup, floating block, product tab, label or size guide | `gutenberg-create-content` |
| Read a target's block tree | `gutenberg-get-content` |
| Change blocks in a non-empty target | `gutenberg-edit-blocks` |
| Fill an empty target or deliberately rebuild all of it | `gutenberg-replace-content` |
| Discover blocks and their attributes | `gutenberg-list-blocks`, then `gutenberg-describe-block` |
| Reuse a demo section or page | `gutenberg-search-templates`, then `gutenberg-get-template` |

An `id` from `search-content`, or a layout's `target_id` from `list-layouts`, is the `target_id` for
Gutenberg abilities.

### Block-write rules

1. **Never rebuild a target to change only part of it.** Read it with `gutenberg-get-content`, then use
   `gutenberg-edit-blocks` and send only the operations and attribute keys that change. This includes
   requests such as "update all texts on the page." Untouched blocks and attributes must be omitted.
2. Use `gutenberg-replace-content` only for an empty target or when the user explicitly asks to
   replace/rebuild the entire target.
3. Author text and content in the attribute named by `gutenberg-describe-block`'s `contentAttribute`.
   Do not hand-write `innerHTML` for content updates. Existing `innerHTML` is round-trip data for content the
   agent is preserving, or verbatim markup supplied for an intentional insert.
4. Only `wd/*` blocks have stable `blockId` values and can be addressed by id. Core blocks pass through but
   cannot be individually updated, deleted, moved, or used as anchors.
5. Omit `attributes.blockId` when authoring or inserting a `wd/*` block; normalization creates it. Use ids
   copied from `gutenberg-get-content` only as operation selectors or anchors—never invent, copy, reuse, or
   send one inside update `attributes`.
6. Pass `expected_content_hash` from the latest `gutenberg-get-content` response when editing. On a
   conflict, re-read the target and rebuild the intended operations against the new tree.

If a deliberate full replacement might touch classic/freeform content outside registered blocks, first read
with `include_raw_content: true`; `block_spec` otherwise cannot represent that content. Do not use full
replacement as a workaround for this limitation.

For a text edit, describe the block if the content attribute is not already known, then update only that
attribute:

```json
{
  "target_id": 123,
  "expected_content_hash": "...",
  "operations": [
    { "op": "update", "blockId": "a1b2", "attributes": { "content": "New heading" } }
  ]
}
```

### Authoring from scratch

1. Use `gutenberg-list-blocks`, then `gutenberg-describe-block` for every block being authored.
2. Fetch referenced `styleGroups` or `composites` with `gutenberg-describe-attribute-group` only when that
   styling is needed.
3. Follow the returned `composition`, `nesting`, `conventions`, examples, units, responsive attributes, and
   data-resolver hints. Do not invent block attributes or site-specific ids.
4. For layouts, pass the relevant type's `block_categories` from `list-layout-types` to
   `gutenberg-list-blocks`.
5. Prefer a library template when it matches the requested section or page. Use its returned `block_spec`;
   demo images are localized during finalization.

### Finalization

Every block-content write is queued and is not live until its batch reaches `finalized`. Only the
**WoodMart → Tools → AI Block Queue** page in the user's browser finalizes it (it runs the real block editor);
neither the server nor the agent can.

1. Queue every change of the batch (`gutenberg-replace-content` / `gutenberg-edit-blocks` with the same
   `batch_id`), then call `gutenberg-finalize-batch` once to mark the draft batch ready.
   `gutenberg-create-content` or `create-layout` with `block_spec` already marks its batch ready.
2. Ask the user to open the AI Block Queue page (`finalization_url`) and keep it open; it picks up ready
   batches by itself and shows a journal with Retry, Cancel and Copy error. Follow the response's
   `user_instruction`.
3. Poll `gutenberg-get-pending-batch` until the batch is `finalized`, `failed` or `canceled` (`last_error` says
   why a batch was canceled). `finalizer_online: false`
   means no AI Block Queue page asked for work in the last minute: remind the user to open it.

- On `failed`, check the items. An item `conflicted` means the target changed after it was queued: re-read the
  target and repeat the edit against the current content. Otherwise read `last_error` and the items'
  `validation_errors` (also in the journal) and fix the change. Either way queue it again: a new change on the
  same target replaces the failed one.

Only one active pending change may exist per target: a second one is refused until the first is finalized.
Cancel an obsolete non-finalized batch with `gutenberg-delete-pending-batch` before queuing a replacement.

Edit an existing layout's blocks against its `target_id`; do not recreate the layout to change its content.
A portfolio project is content of post type `portfolio` (needs the Portfolio theme option); edit it like a page.

## Elementor

| Intent | Start with |
|--------|------------|
| Create a page, post, portfolio project, HTML block, popup, floating block or product tab | `elementor-create-content` |
| Read an element tree | `elementor-get-content` (skeleton; `element_id` for one element in full) |
| Discover elements and their controls | `elementor-list-elements` → `elementor-describe-element` |
| Change part of a page | `elementor-edit-elements` (operations + `expected_content_hash`) |
| Fill an empty target or deliberately rebuild all of it | `elementor-replace-content` |
| Document settings (popup size and style, slide `wd_*`, page settings) | `elementor-get-document-settings` → `elementor-update-document-settings` |
| Reuse a demo section or page | `elementor-search-templates` → as is: pass `template_id` to the write; to change it first (texts, number of items): `elementor-get-template`, edit the tree, send it as `elements` |
| Clear Elementor's CSS files and cache (Tools > Clear Files & Data), e.g. when a page renders stale styles | `elementor-clear-cache` |
| Elementor → Settings and Features; the kit id for Site Settings | `elementor-get-settings` → `elementor-update-settings` |

Writes are synchronous: each one validates the whole tree and saves immediately; an invalid tree saves
nothing. `replace-content` needs `allow_content_loss` to drop existing content; set it only if the user asked.

1. **Describe before authoring.** `elementor-describe-element` lists an element's controls (drill down with
   `tab`, `section`, `names`, `prefix`). Never invent element names or control keys. Responsive values use
   the `_tablet` / `_mobile` suffixes it reports (ask for the base name; the suffixed keys are not listed).
   Tree shape: `{"elType":"container","settings":{…},"elements":[{"elType":"widget","widgetType":"wd_title","settings":{…}}]}`.
2. **Build with `wd_*` widgets.** When `elementor-list-elements` has `e-flexbox`, lay out with V4 containers
   (see "Elementor V4 containers"). With a V3 `container`: an empty `flex_direction` renders as a column; set
   `flex_direction: "row"` for side-by-side items (and `flex_direction_mobile: "column"` to stack them). Items in a
   row do not share the width equally; for equal columns use `container_type: "grid"` with `grid_columns_grid`
   (defaults: 3 columns and 2 rows). On the WoodMart standard the kit gives containers no padding; on a site that
   is not set up yet, run the last section first. Set `padding` only on containers with a background or border.
   `wd_infobox` takes an image or text as its icon, not an icon font.
3. **Set every visible text.** Unset text controls render placeholder defaults (e.g. "Infobox subtitle text",
   "Read more"); give each one a value or `""`.
4. **Use the control's own values.** A switcher takes its `on_value` from `describe-element` (often `"1"`), not
   a generic `"yes"`. A key that is not a control comes back as `wd_ai_unknown_setting`: on classic elements Elementor
   stores it but ignores it, so fix it from the "Did you mean" hint.
   Copy the shape of a value from the control's `default` in `describe-element` (e.g. `padding`:
   `{"unit":"px","top":"0","right":"0","bottom":"35","left":"0","isLinked":false}`).
5. **Write tiny.** Edit only the elements and keys that change, by `elementId`. Omit `id` on new elements.
   Every `elementor-edit-elements` operation needs `op`: `{"op":"update","elementId":"abc1234","settings":{…}}`,
   `{"op":"insert","element":{…},"position":"last-child","anchor":"<parent id>"}`, `{"op":"delete","elementId":"…"}`,
   `{"op":"move","elementId":"…","position":"after","anchor":"…"}`. `elementor-get-content` takes one `element_id`
   (a string, not a list): call it once per element.
6. Popup, floating block and product tab display conditions are not document settings: use
   `set-display-conditions`. A layout made with `create-layout` without `predefined_name` starts empty: fill it with
   `elementor-replace-content`. A `wd_slider` widget takes the slider's slug; slide content is written with
   `save-slide` `elements`.

### Elementor V4 containers

Lay out with V4 containers (`e-flexbox`, `e-grid`); everything inside stays V3 (`wd_*` and classic widgets),
except the V4 off-canvas element. Style V4 containers with local `styles` (shape in `elementor-describe-element`).

1. **Stacked widgets:** `e-flexbox` with `flex-direction` `column`. A default `e-flexbox` is a row and squeezes V3
   widgets to a few pixels.
2. **Spacing:** V4 containers have no gap: set `gap` 20px on the `e-flexbox`. `gap` does nothing on `e-div-block`.
3. **Columns:** equal widths: `e-grid`, one column on mobile (`grid-template-columns` `1fr`); widths by content:
   row `e-flexbox`, `_flex_size: "grow"` on each widget, `flex-direction` `column` on mobile.
4. **Full-width background, content at site width:** outer `e-flexbox` with `wd_stretch` `stretch`, inner
   `e-flexbox` with `wd_stretch` `boxed`. Nothing else: no max-width, padding or justify-content.
5. **WoodMart props** (strings, not booleans): `e-flexbox` has `wd_stretch` and `wd_color_scheme`; `e-div-block`
   only `wd_color_scheme`; `e-grid` none. Everything else is Elementor's own styles.
6. **Use a V3 `container` instead** when the wrapper needs a WoodMart animation, sticky, parallax or collapsible
   content; V4 containers have none of them. Widgets keep their own animations.
7. **Off-canvas sidebar:** `wd-atomic-off-canvas` with `wd-atomic-sidebar` and `wd-atomic-off-canvas-content`;
   open it on mobile with `wd_builder_off_canvas_column_btn`. Put the sidebar widgets in the sidebar's e-flexbox.
   V4 elements with a structure (tabs, accordion, off-canvas, background video) written without children get the
   editor's structure, and their locked children are completed; `elementor-describe-element` shows it as
   `default_children`. Then change texts by id.
8. **Deprecated widgets:** not `wd_banner_carousel`, `wd_infobox_carousel` or `wd_tabs`; use
   `wd_nested_carousel` with `wd_banner`, and `wd_tabs_nested`.
9. **No global classes; do not create or change variables** (Theme Settings writes the theme ones on save).
   Theme variables: `theme-primary-color`, `theme-alternative-color`, `theme-border-radius`. V4 styles:
   `{"$$type":"global-color-variable","value":"<e-gv id>"}` for colors, `{"$$type":"woodmart-size-variable","value":"<e-gv id>"}`
   for the radius; ids from `elementor-list-variables`. V3 widgets: `"__globals__":{"<color control>":"globals/colors?id=<v3_global_id>"}`.
   `elementor-get-settings` `editor` tells whether V4 is on.

**Converting a V3 page to V4** (only when asked): write a draft copy, keep the original. Replace each V3 `container`
with an `e-flexbox` (`e-grid` for a grid container) by the rules above: `flex_direction` → `flex-direction`,
`flex_gap` → `gap`, padding, margin and background → styles, `_tablet` / `_mobile` values → tablet / mobile variants,
`wd_section_stretch` → rule 4. Keep a V3 `container` where rule 6 applies, or for shape dividers or a YouTube
background video. A child with a width and no mobile width gets `width` 100% (or a `column` parent) on mobile: V3 did
it by itself. Elementor widgets with a V4 twin may move to it (heading → `e-heading`, text-editor → `e-paragraph`,
button → `e-button`, image → `e-image`, divider → `e-divider`, nested tabs/accordion → `e-tabs`/`e-accordion`);
carry their look over in styles: heading `line-height`, button padding and font size, divider spacing and color,
tab and accordion colors (`e--selected` for the active tab). `wd_*` widgets and widgets without a twin stay V3. V4
widgets inside a V3 `container` with a background overlay need `position: relative` and `z-index: 1`. Compare the
copy with the original at desktop, tablet and mobile widths and list what did not carry over; without the front end,
list the places you could not check.

### Elementor site without the WoodMart demo import

The demo import sets Elementor and WooCommerce up the WoodMart way. A site without it shows a non-empty
`missing_theme_values` in `elementor-get-settings`; set it up before authoring content. Every step changes the
whole site: tell the user what will change and get a yes first.

1. Pass `missing_theme_values` from `elementor-get-settings` as the input of `elementor-update-settings`, as is
   (post types Elementor edits, default colors and fonts off, the Container feature on).
2. Kit (Site Settings): `elementor-update-document-settings` with `target_id` = `kit_id` from
   `elementor-get-settings`: `container_padding` `{"unit":"px","top":"0","right":"0","bottom":"0","left":"0","isLinked":true}`,
   `viewport_mobile` `768.98`, and `system_typography` with the same entries (`_id`, `title`,
   `typography_typography`, `typography_font_weight`) but without `typography_font_family`, so texts use the theme
   fonts. Then `elementor-clear-cache`, or the old mobile breakpoint stays in the CSS.
3. `update-customizer-settings`: `woocommerce_single_image_width` `1200`, `woocommerce_thumbnail_image_width` `600`.
4. Pages: the blog page with `update-site-settings` (`show_on_front` `page`, `page_for_posts`); the shop page with
   `update-store-settings` (group `products`, `woocommerce_shop_page_id`); registration on My account with
   `update-store-settings` (group `account`, `woocommerce_enable_myaccount_registration`); cart, checkout and my
   account without sidebar with `update-metaboxes` (`_woodmart_main_layout` `full-width`). If the site has no blog
   page (or no home page for `show_on_front` `page`), ask the user whether to create it; do not pick another page.
5. Ask the user to open wp-admin once: the theme then gives Elementor's new div and section blocks no padding.
   The import also removes the Uncategorized product category; MCP cannot delete it, so tell the user.
