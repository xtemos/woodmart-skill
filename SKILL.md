---
name: woodmart-mcp
description: "Use for WoodMart site work through the woodmart MCP abilities (woodmart/*): finding or creating content, editing Gutenberg blocks, changing theme settings or header logos, and creating or assigning WoodMart layouts. Covers the queued block-finalization workflow and the safety rules that coordinate multiple abilities."
---

# Working with WoodMart via MCP abilities

## Use live ability documentation

Everything runs through the woodmart MCP adapter. Use `mcp-adapter-discover-abilities` for
the current ability list and `mcp-adapter-get-ability-info` before the first use of a write
ability, or whenever its parameters are uncertain. Treat the live description, input schema,
response fields, warnings, and `user_instruction` as authoritative; do not rely on remembered
parameter lists.

Use `mcp-adapter-execute-ability` with `ability_name` and `parameters` to execute an ability.

If the woodmart MCP is unavailable or every call fails, stop and tell the user the server name and error.
Do not continue through wp-cli, PHP, SQL or hand-built REST calls.

## Rules for every task

1. **Reuse before building.** Look for what the site already has (a similar section or HTML block) and for
   a WoodMart demo template (`gutenberg-search-templates`) before authoring from scratch. A close template
   with other texts is still the better start: insert it and change only the texts (its demo images are
   copied to the media library). Prefer WoodMart blocks (`wd/*`) over core ones when both fit.
2. **Read, then write only the difference.** Read the current state right before a write and send only what
   changes, with the hash that read returned. Where a description says a list is complete or replaces the old
   one (layout conditions), send every entry that must remain.
   **Never store a default:**
   - Before writing, skip any value equal to the block's `default` or to what the theme and Theme Settings
     already give.
   - Remove with `removeAttributes`, never `""` or `0`.
   - After writing, compare `getComputedStyle` with and without each attribute you set, at desktop, tablet and
     mobile. No change → remove it. A change → keep it, even when it looks like a default (e.g. `align: start`).
     This check finds duplicates too: an attribute that repeats what the block's predefined style (a style or
     design option, a theme class) already gives changes nothing, so remove it.
3. **Report what really happened.** Relay `warnings` (a moved URL, a preset override, a setting with no
   effect). List what you could not do as "where → field → value" for the user. Re-read after writing. Check
   visible changes on the front end at desktop, tablet and mobile widths when you can open the site;
   otherwise ask the user to check them.

## Design: match the site

1. **Take values from the site, not from taste.** Before authoring a section, read a similar section of the
   site and reuse its spacing, heading sizes, colors and radius; when the site has none, use the closest
   WoodMart template's values.
2. **Inherit globals.** Fonts, colors and rounding come from Theme Settings. Use theme color variables
   (`var(--wd-primary-color)`, or the block's color-variable attribute) instead of hex copies; set a value
   only where it differs from the global one. Page background: the site's main color in every Theme Settings
   page background (single product included, no preset just to recolor one page); a section gets its own
   background only where it differs, never to repeat the page color.
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
   `create-content`; then pick them in Theme Settings (`wishlist_page`, `compare_page`) with
   `update-theme-setting`.

## Routing

| Intent | Start with |
|--------|------------|
| Find a page, post, product, portfolio project, or HTML block | `woodmart/search-content` |
| Create a page, post, portfolio project, or HTML block | `woodmart/create-content` |
| Rename, publish, re-slug, or re-parent content | `woodmart/update-content` |
| Read a target's block tree | `woodmart/gutenberg-get-content` (a section's name is `attributes.metadata.name`) |
| Change blocks in a non-empty target | `woodmart/gutenberg-edit-blocks` |
| Fill an empty target or deliberately rebuild all of it | `woodmart/gutenberg-add-pending-change` |
| Discover blocks and their attributes | `woodmart/gutenberg-list-blocks`, then `woodmart/gutenberg-describe-block` |
| Reuse a demo section or page | `woodmart/gutenberg-search-templates`, then `woodmart/gutenberg-get-template` |
| Read or change a theme setting | `woodmart/list-theme-settings`, then `woodmart/get-theme-setting` |
| Replace Header Builder logos | `woodmart/replace-header-logo` |
| Discover, create, or assign layouts | `woodmart/list-layout-types`, then the relevant layout ability |

An `id` from `search-content`, or a layout's `target_id` from `list-layouts`, is the
`target_id` for Gutenberg abilities.

## Block-write rules

These rules are non-negotiable because violating them can silently lose content or styling.

1. **Never rebuild a target to change only part of it.** Read it with
   `gutenberg-get-content`, then use `gutenberg-edit-blocks` and send only the operations and
   attribute keys that change. This includes requests such as “update all texts on the page.”
   Untouched blocks and attributes must be omitted.
2. Use `gutenberg-add-pending-change` only for an empty target or when the user explicitly asks
   to replace/rebuild the entire target.
3. Author text and content in the attribute named by `gutenberg-describe-block`'s
   `contentAttribute`. Do not hand-write `innerHTML` for content updates. Existing `innerHTML`
   is round-trip data for content the agent is preserving, or verbatim markup supplied for an
   intentional insert.
4. Only `wd/*` blocks have stable `blockId` values and can be addressed by id. Core blocks pass
   through but cannot be individually updated, deleted, moved, or used as anchors.
5. Omit `attributes.blockId` when authoring or inserting a `wd/*` block; normalization creates
   it. Use ids copied from `gutenberg-get-content` only as operation selectors or anchors—never
   invent, copy, reuse, or send one inside update `attributes`.
6. Pass `expected_content_hash` from the latest `gutenberg-get-content` response when editing.
   On a conflict, re-read the target and rebuild the intended operations against the new tree.
7. Name every top-level section you add (see "Section names").

If a deliberate full replacement might touch classic/freeform content outside registered
blocks, first read with `include_raw_content: true`; `block_spec` otherwise cannot represent
that content. Do not use full replacement as a workaround for this limitation.

**qTranslate-XT pages** are the exception. All languages share one `post_content`, separated by
`[:ua]`, `[:ru]`, `[:]` tags between blocks. `gutenberg-edit-blocks` cannot keep these tags, and
`block_spec` shows the blocks of every language without them. Read with `include_raw_content: true`
to see which blocks belong to which language. Change the page only with
`gutenberg-add-pending-change` and `allow_raw_html: true`, sending the whole page in every language.
Send each tag as its own top-level block `{"name": "core/freeform", "attributes": {"content": "[:ua]"}}`,
and close the last language with `[:]`. Keep the blocks you do not change as they are. Without the
tags, all languages merge into one.

This server edits block (Gutenberg) content only. Pages built with Elementor or WPBakery, and
pages still holding classic-editor HTML (`gutenberg-edit-blocks` refuses them with
`wd_ai_content_outside_blocks`), are out of scope for block edits: tell the user and replace such
a page only when the user asks for a full replacement. An Elementor or WPBakery page keeps
showing its builder content after a block replacement until it is switched back to the
WordPress editor; say so. Theme settings, layouts and their conditions, the header logo and post
metadata work the same on every page builder.

### Section names

Name every top-level section you add in `attributes.metadata.name` (the List View name; `gutenberg-get-content`
returns it, so find a section by it); do not rename existing ones unless the user asks. If the user asks not to name
or rename sections, do not, and leave the `wd_ai_unnamed_sections` warning without action.

A name is English, sentence case (only the first word capitalised): the purpose (1-3 words: Hero, Categories, Products,
Banner, Features, About, Testimonials, Brands, Blog, Newsletter, Call to action, FAQ, Contact) plus the block's role, so
a section is never confused with an element: `wd/section` (full-width background) → "Full width <purpose> section",
`wd/row` → "<Purpose> row", `wd/container` → "<Purpose> container", any other top-level block that holds blocks →
"<Purpose> <block type>" (Hero slider, Categories carousel, Gallery tabs, Contact accordion). The same kind of section
gets the same name on every page; rename it when its block changes. Name only the top level, and keep the names the
theme sets by itself (`wd/cover` "Banner content", `wd/infobox` "Infobox content", `wd/toggle` "Toggle header").

`wd_ai_unnamed_sections` lists top-level sections without a name. Name sections before queuing; if the batch is not
finalized yet, cancel it with `gutenberg-delete-pending-batch` and queue the change again with names. On a finalized
page use a `gutenberg-edit-blocks` `update` of `attributes.metadata`; `metadata` is replaced as a whole, so send the
block's existing `metadata` keys plus `name`.

## Authoring from scratch

1. Use `gutenberg-list-blocks`, then `gutenberg-describe-block` for every block being authored.
2. Fetch referenced `styleGroups` or `composites` with
   `gutenberg-describe-attribute-group` only when that styling is needed.
3. Follow the returned `composition`, `nesting`, `conventions`, examples, units, responsive
   attributes, and data-resolver hints. Do not invent block attributes or site-specific ids.
4. For layouts, pass the relevant type's `block_categories` from `list-layout-types` to
   `gutenberg-list-blocks`.
5. `wd/testimonial`, `wd/team-member` and `wd/timeline-item` keep their content in locked inner
   blocks that `gutenberg-describe-block` does not show: copy the structure from a library template
   (`gutenberg-search-templates`, then `gutenberg-get-template`) or a page that has the block, and
   change only the texts.
6. Prefer a library template when it matches the requested section or page. Use its returned
   `block_spec`; demo images are localized during finalization.

## Finalization

Every block-content write is queued and is not live until its batch reaches `finalized`.
Metadata, theme-setting, header-logo, predefined-layout, and layout-condition writes do not use
the block queue.

1. Follow the queue response's `user_instruction`. Ask the user to keep
   **WoodMart → Tools → AI Block Queue** open in wp-admin.
2. Call `gutenberg-enable-finalization` when the response says the batch is still a draft.
   `create-content` or `create-layout` with `block_spec` already marks its batch ready, so do not
   enable it again.
3. Poll `gutenberg-get-pending-batch` until the batch is `finalized`, `failed` or `canceled`
   (`last_error` says why a batch was canceled).
4. On `failed`, check the items. An item `conflicted` means the target changed after it was
   queued: re-read the target and queue the edit again against the current content. Otherwise
   read `last_error` and the items' `validation_errors`, fix the change and queue it again. The
   new change replaces the failed one; no need to delete the failed batch first.
5. A batch left `running` by a closed queue page is picked up again after 3 minutes while the
   page is open. A target moved to the Trash or deleted cancels its batch.

Only one active pending change may exist per target. Cancel an obsolete batch that is not
finalized or failed with `gutenberg-delete-pending-batch` before queuing a replacement.

## Immediate writes and reporting

### Content metadata

`update-content` changes metadata immediately and never changes blocks.

- Publishing is outward-facing. Set `status: "publish"` only when the user requested it or
  explicitly confirmed it.
- Changing the slug or parent of published content moves its public URL. Obtain confirmation
  unless the user explicitly requested that URL change.
- Read and relay the response's `warnings`, actual `after.slug`, and resulting URL. Published
  hierarchical pages do not receive WordPress's old-slug redirect; their old URL can become a
  404.
- There is no trash/delete-content ability. Do not simulate deletion by blanking block content.

### Theme settings and logos

- Find a theme-setting id with `list-theme-settings`; do not guess it.
- Read the full descriptor with `get-theme-setting` immediately before writing. Honor
  `writable`, `value_format`, `options`, `current_value`, and `has_preset_overrides`.
- `update-theme-setting` writes the base layer. A preset override may still determine the
  visible value; report that instead of retrying blindly.
- Typography and idle/hover colors merge: send only the keys or states that change; `null`
  removes one. A typography list changes fonts by position and keeps the others; `null` at a
  position removes that font. Values are checked against the descriptor (listed options,
  range bounds, color formats), and JavaScript editors are read-only.
- `replace-header-logo` requires an existing media-library image `attachment_id` and updates
  every saved header. Read the per-header statuses and `totals`; report skipped or failed
  headers rather than claiming blanket success.

### Layouts

- Start with `list-layout-types`. Use its live `type`, `block_categories`, `predefined`, and
  `condition_types` data rather than maintaining local enums.
- Edit an existing layout's blocks with Gutenberg abilities against its `target_id`; do not
  recreate the layout merely to change its content.
- `set-layout-conditions` replaces the complete condition set. Send every condition that must
  remain; `[]` unassigns the layout.
- Build each condition using the selected layout type's `condition_types` entry and its
  `query_kind` (`none`, `scalar`, `array`, or `number_range`).
- `is_assigned` is request-context-dependent over MCP. Inspect stored `conditions`; do not
  rewrite them solely because `is_assigned` is false.

## Portfolio and site-specific ids

A portfolio project is content of post type `portfolio`; find/create it and edit its Gutenberg
blocks like a page. `single_portfolio` and `portfolio_archive` are layouts controlling how
projects or archives render. Portfolio content requires the Portfolio theme option to be enabled.

Never invent product, category, taxonomy, media, post, or template ids. Resolve them with the
relevant live query/search ability, or use a dynamic block query when explicit ids are unnecessary.
