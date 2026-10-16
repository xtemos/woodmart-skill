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
| Page/post/product/term options (page title, sidebar, header off, product layout…), popup and rule style fields | `get-metaboxes` → `update-metaboxes` |
| Theme setting for the whole site | `list-theme-settings` → `get-theme-setting` → `update-theme-setting(s)` |
| Theme setting only on some pages | preset: `list-theme-settings-presets` → `save-theme-settings-preset` → `update-theme-settings` with `preset_id` |
| Back up Theme Settings before many changes; undo them | `create-theme-settings-backup`; later `list-theme-settings-backups` → `get-theme-settings-backup` (what differs now) → `update-theme-settings` with the `backup_value` of the settings to restore |
| Header, logo, header elements (desktop and mobile columns hold separate elements, e.g. two logos) | `list-headers` → `get-header` → `describe-header-element` → `edit-header` |
| Menu items, mega menu (also the menu a header shows) | `list-menus` → `get-menu` → `describe-menu-item-settings` → `edit-menu`; `create-menu` |
| Media | `list-media` first; `upload-media` (a URL) or `request-media-upload` (local files); `update-media` for alt/title |
| Categories, tags, attributes, swatches (a value becomes a swatch through its term's `swatch_text` / `swatch_color` / `swatch_image_id`) | `list-taxonomies` → `list-terms` → `create-term` / `update-term` / `set-object-terms`; `create-product-attribute` |
| Products (a variable product's prices live in its variations) | `create-simple-product` / `create-variable-product`, then `update-product` (WooCommerce fields) and `update-product-data` (WoodMart fields: size guide, 360, videos, bundles, variation images) |
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
   `create-content`; then pick them in Theme Settings (`wishlist_page`, `compare_page`).

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
| Create a page, post, portfolio project, HTML block, popup, floating block, product tab, label or size guide | `create-content` |
| Read a target's block tree | `gutenberg-get-content` |
| Change blocks in a non-empty target | `gutenberg-edit-blocks` |
| Fill an empty target or deliberately rebuild all of it | `gutenberg-add-pending-change` |
| Discover blocks and their attributes | `gutenberg-list-blocks`, then `gutenberg-describe-block` |
| Reuse a demo section or page | `gutenberg-search-templates`, then `gutenberg-get-template` |

An `id` from `search-content`, or a layout's `target_id` from `list-layouts`, is the `target_id` for
Gutenberg abilities.

### Block-write rules

These rules are non-negotiable because violating them can silently lose content or styling.

1. **Never rebuild a target to change only part of it.** Read it with `gutenberg-get-content`, then use
   `gutenberg-edit-blocks` and send only the operations and attribute keys that change. This includes
   requests such as "update all texts on the page." Untouched blocks and attributes must be omitted.
2. Use `gutenberg-add-pending-change` only for an empty target or when the user explicitly asks to
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

```text
Existing blocks?
├── no  → create-content with block_spec, or gutenberg-add-pending-change
└── yes → Explicit full rebuild requested?
          ├── no  → gutenberg-edit-blocks
          └── yes → gutenberg-add-pending-change
```

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

Every block-content write is queued and is not live until its batch reaches `finalized`. No other ability
uses the block queue. A batch is finalized in one of three ways, in this order of preference:

1. **On the server.** When the site has Node, `gutenberg-enable-finalization` (or `create-content` /
   `create-layout` with `block_spec`) finalizes the batch during the call: the response has
   `batch_status: finalized` and `finalized_by: server`. Nothing else to do.
2. **From your computer.** When the batch comes back `ready` with `local_serializer.command`, and you can run
   commands and have Node 18 or newer, run that command as is:
   `curl -sS -H "X-WD-AI-Token: <token>" "<site>/wp-json/woodmart-mcp/v1/gb-queue/local/runner" | node`.
   It serializes the blocks on your computer and prints `Batch #N finalized.` (exit 0), or the reason it
   could not. The token is one-time and expires in 15 minutes; `gutenberg-get-pending-batch` returns the
   same command while it is alive and a new one afterwards. Do not edit the command or reuse a token.
3. **In the browser.** Otherwise ask the user to open **WoodMart → Tools → AI Block Queue** in wp-admin
   and keep it open; the page finalizes the batch and shows a journal with Retry, Cancel and Copy error.

In every case:

- Follow the queue response's `user_instruction`; it already says which way applies.
- Call `gutenberg-enable-finalization` only when the batch is still a draft, after every change of the
  batch is queued. `create-content` or `create-layout` with `block_spec` already marks its batch ready.
- Poll `gutenberg-get-pending-batch` until `finalized`, `failed`, or `conflicted`; `finalizer_online`
  tells whether an AI Block Queue page is open, `server_serializer` whether the site has Node.
- On `failed`, read `last_error` (also shown in the journal), fix the change and queue it again.
- On `conflicted`, re-read the target and repeat the edit against the current content.

Only one active pending change may exist per target. Cancel an obsolete non-finalized batch with
`gutenberg-delete-pending-batch` before queuing a replacement.

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

Writes are synchronous: each one validates the whole tree and saves immediately; an invalid tree saves
nothing. `replace-content` needs `allow_content_loss` to drop existing content; set it only if the user asked.

1. **Describe before authoring.** `elementor-describe-element` lists an element's controls (drill down with
   `tab`, `section`, `names`, `prefix`). Never invent element names or control keys. Responsive values use
   the `_tablet` / `_mobile` suffixes it reports (ask for the base name; the suffixed keys are not listed).
   Tree shape: `{"elType":"container","settings":{…},"elements":[{"elType":"widget","widgetType":"wd_title","settings":{…}}]}`.
2. **Build with `container` and `wd_*` widgets.** An empty `flex_direction` renders as a column: set
   `flex_direction: "row"` for side-by-side items (and `flex_direction_mobile: "column"` to stack them).
   Items in a row do not share the width equally by themselves; for equal columns use `container_type: "grid"` with
   `grid_columns_grid` (defaults: 3 columns and 2 rows). Containers get the kit's default padding (10px) unless
   `padding` is set. `wd_infobox` takes an image or text as its icon, not an icon font.
3. **Set every visible text.** Unset text controls render placeholder defaults (e.g. "Infobox subtitle text",
   "Read more"); give each one a value or `""`.
4. **Use the control's own values.** A switcher takes its `on_value` from `describe-element` (often `"1"`), not
   a generic `"yes"`. A key that is not a control comes back as `wd_ai_unknown_setting`: on classic elements Elementor
   stores it but ignores it, so fix it from the "Did you mean" hint.
5. **Write tiny.** Edit only the elements and keys that change, by `elementId`. Omit `id` on new elements.
6. Popup, floating block and product tab display conditions are not document settings: use
   `set-display-conditions`. A layout made with `create-layout` without `predefined_name` starts empty: fill it with
   `elementor-replace-content`. A `wd_slider` widget takes the slider's slug; slide content is written with
   `save-slide` `elements`.
