---
name: woodmart-mcp
description: Use for any WoodMart site work through the wood MCP abilities (woodmart/*) — editing/building Gutenberg page & block content, finding, creating, renaming, publishing or re-slugging posts/pages/portfolio projects/HTML blocks, reading and changing WoodMart theme settings (colors, typography, toggles), and creating/targeting WoodMart layout templates (single product, shop archive, cart, checkout, my account, blog, portfolio) with assignment conditions. Covers gutenberg-*, search-content, create-content, update-content, *-theme-setting(s), and *-layout(s) abilities, plus finalizing queued block changes so they go live.
---

# Working with WoodMart via MCP abilities

## How to call

Everything runs through the wood MCP adapter: `mcp-adapter-execute-ability` with
`ability_name` + `parameters`. Use `mcp-adapter-get-ability-info` if you need a live schema.

There are four domains. Pick the one that matches the request:

| Domain | You want to… | Goes live |
|--------|--------------|-----------|
| **Content** | find, create, rename/publish a page/post/HTML block | immediately (block content needs finalization) |
| **Gutenberg** | read/edit/author the blocks inside a target | **only after finalization** |
| **Theme settings** | change colors, typography, toggles site-wide | immediately |
| **Layouts** | create/target a template (product, shop, cart, checkout, …) | immediately (block content needs finalization) |

Three rules govern all block work. Rule 1 is the one that gets broken most often:

1. **NEVER rebuild a page to change part of it.** If the target already has blocks and the
   request touches *some* of them — change the texts, swap an image, fix a heading, delete a
   section, add a button — you use `gutenberg-edit-blocks` and send **only the operations that
   change**. Resending the whole page through `gutenberg-add-pending-change` is not a "safe
   fallback": it is a rewrite of every block from your own reconstruction of them, and every
   attribute you fail to reproduce exactly is silently lost. It also costs enormously more
   tokens, and long block trees are exactly where that reconstruction starts drifting.
   `gutenberg-add-pending-change` is for an **empty target** or a **deliberate full rebuild the
   user asked for** — nothing else. If you catch yourself about to resend blocks you were not
   asked to change, stop and switch to `edit-blocks`.
2. **Author content in block ATTRIBUTES, never in hand-written innerHTML.** The editor's
   `save()` generates the markup. innerHTML is only for preserving a block you are NOT
   changing (verbatim round-trip).
3. **Queued block changes are NOT live until finalized.** Every block write goes into a batch
   that must be finalized through the AI Block Queue admin page. Skipping this = nothing changed.

Only `wd/*` blocks carry a stable `blockId` and are individually addressable. Core blocks
pass through untouched but cannot be targeted by id.

### Which write ability?

```
Does the target already have blocks?
├── no  → gutenberg-add-pending-change (or create-content with block_spec)
└── yes → Did the user ask to rebuild/replace the entire page?
          ├── no  → gutenberg-edit-blocks   ← almost always this
          └── yes → gutenberg-add-pending-change
```

## All abilities

**Content**

| Task | Ability |
|------|---------|
| Find a post/page/product/portfolio project/HTML block by title (get its id) | `woodmart/search-content` |
| Create a new post/page/portfolio project/HTML block (empty or with content) | `woodmart/create-content` |
| Rename / re-slug / publish / re-parent an existing one | `woodmart/update-content` |

**Gutenberg blocks** (work against any `target_id`: a page, an HTML block, or a layout)

| Task | Ability |
|------|---------|
| Read a target's blocks + content_hash + blockIds | `woodmart/gutenberg-get-content` |
| Discover what blocks exist | `woodmart/gutenberg-list-blocks` |
| Learn one block's attributes + example | `woodmart/gutenberg-describe-block` |
| Expand a style group / advanced set | `woodmart/gutenberg-describe-attribute-group` |
| Find a pre-built section/page to reuse | `woodmart/gutenberg-search-templates` |
| Get a library template's block_spec | `woodmart/gutenberg-get-template` |
| **Change anything on a page that already has blocks** (update/delete/insert/move) | `woodmart/gutenberg-edit-blocks` |
| Fill an **empty** target, or a full rebuild the user explicitly asked for | `woodmart/gutenberg-add-pending-change` — *last resort, see rule 1* |
| Mark a batch ready to go live | `woodmart/gutenberg-enable-finalization` |
| Poll a batch until it is live | `woodmart/gutenberg-get-pending-batch` |
| Cancel a queued batch | `woodmart/gutenberg-delete-pending-batch` |

**Theme settings**

| Task | Ability |
|------|---------|
| Search/browse settings (`search`, `section`, `per_page`, `page`) | `woodmart/list-theme-settings` |
| Read one setting's full metadata + current value (`id`) | `woodmart/get-theme-setting` |
| Change one setting (`id`, `value`) | `woodmart/update-theme-setting` |
| Swap the logo in **every** header (`attachment_id`) | `woodmart/replace-header-logo` |

**Layouts**

| Task | Ability |
|------|---------|
| Entry point: list layout types, their block_category + predefined templates | `woodmart/list-layout-types` |
| List existing layouts (`type`, `per_page`, `page`) → gives `target_id` | `woodmart/list-layouts` |
| Create a layout (empty / predefined / authored) | `woodmart/create-layout` |
| Replace a layout's assignment conditions | `woodmart/set-layout-conditions` |

## Workflow A — find a target, or create one

Do this first whenever you don't already have an id.

1. **Existing target:** `search-content` (`query`, optional `post_types`/`status`/`limit`) →
   pick a result's `id`; that is the `target_id` for every `gutenberg-*` ability.
   Searchable types: `page`, `post`, `product`, `portfolio`, `cms_block` — omit `post_types`
   to search all of them.
2. **New page/post/portfolio project/HTML block:** `create-content` (`title`, optional
   `post_type` [`page`|`post`|`portfolio`|`cms_block`, default `page`], `status`, `slug`,
   `parent_id`).
   - No `block_spec` → returns `{id, post_type, post_status, edit_url}` empty; fill it later
     with Workflow B/C/D.
   - With `block_spec` → queues the content **and auto-marks it ready**, so skip
     `gutenberg-enable-finalization` and go straight to polling (Workflow E).

Two of these types are conditional: `product` needs WooCommerce active, and `portfolio` needs
the Portfolio theme option enabled. Asking for one that isn't registered returns an explicit
error naming the reason — it does **not** silently fall back to the other types.

### Renaming, publishing, moving

`update-content` (`target_id`, plus any of `title`, `slug`, `status`, `parent_id`) changes a
post's **metadata**, never its block content. It applies **immediately** — no batch, no queue,
no AI Block Queue page. Block content still goes through Workflow B/C and finalization; the two
are separate and can be done in either order.

- Works on `page`, `post`, `portfolio`, `product`, `cms_block`, and `woodmart_layout`. Any other
  post type is rejected.

**Two of these inputs act on the public site. Confirm with the user before either, unless they
asked for it explicitly:**

- **`status: "publish"`** makes the content publicly visible. Setting `draft` again unpublishes it.
- **`slug` (and `parent_id`) on already-published content** moves its public URL. What happens to
  the old URL depends on the post type, and the difference matters:
  - `post`, `product`, `portfolio` (non-hierarchical) → WordPress saves the old slug, so the old
    URL **301-redirects** to the new one. Verified: the old URL returns 301.
  - **`page` (hierarchical) → there is NO redirect. The old URL starts returning 404**, and every
    existing link, bookmark, and search-engine result pointing at it is broken until someone adds
    a redirect by hand. This is WordPress core behaviour (`wp_check_for_changed_slugs()` skips
    hierarchical types), not something the ability can fix. Verified: the old URL returns 404.
  - Renaming the slug of a *draft* is harmless — there is no live URL yet.

The response carries a **`warnings`** array that says exactly which of these happened, with the
old and new URL. It is empty when nothing public moved. Read it and relay it to the user; do not
report a slug change as a clean success when it broke a live URL.

- `slug` may collide, in which case WordPress appends a suffix. The response's `after.slug`
  reports what actually landed — read it rather than assuming your value was taken.
- `parent_id` only applies to hierarchical types (`page`), must be the same post type, and
  cannot be the post itself or one of its descendants. `0` removes the parent.
- The response returns `before` (only the fields you changed) and `after` (the full resulting
  state), so you can report the change precisely.
- Deleting/trashing content is **not** available — there is no delete ability. Say so rather
  than trying to fake it (e.g. do not "delete" by blanking the blocks).

## Workflow B — edit an existing page (the default; see rule 1)

Use this for **every** change to a target that already has blocks, however large the change
feels. "The user asked me to update all the texts on the page" is still Workflow B — it is one
`update` op per text block, not a rebuild. Ten `update` ops are cheaper, safer, and more
accurate than one whole-page `block_spec`, because everything you did not send stays exactly
as it was instead of being reconstructed by you.

1. `gutenberg-get-content` (`target_id`) → note `content_hash` and the `blockId`s you need.
   Pass `include_raw_content: true` first if the target may hold classic-editor/freeform HTML
   that lives outside any registered block — `block_spec` omits it, and a replace-content
   change would drop it silently.
2. `gutenberg-edit-blocks` with `target_id`, optional `expected_content_hash` (the hash you just
   read — guards against stale edits), and `operations`:
   - `{op:"update", blockId, attributes?, removeAttributes?, innerBlocks?, innerHTML?, name?}` —
     omit a field to keep it. `attributes` is **MERGED**: send only the keys you add or change;
     all other existing attributes are preserved — do NOT resend unchanged styling. Reset an
     attribute to its default by listing it in `removeAttributes` or setting it to `null`.
     `blockId` is always preserved.
   - **Changing text: set ONLY the content attribute. NEVER send innerHTML for a text update.**
     The queue regenerates markup from attributes via the block's `save()`. Sending innerHTML
     for a text edit is forbidden — it bloats the call, makes you guess theme markup/classes,
     and the stale markup can override your new text. This applies even if it feels "safer".
     innerHTML is ONLY for an insert that pastes a block verbatim.
   - `{op:"delete", blockId}`
   - `{op:"insert", block:{name, attributes}, position, anchor?}`
   - `{op:"move", blockId, position, anchor?}`
   - `position`: `before`|`after`|`first-child`|`last-child`. `anchor` is a blockId (required
     for before/after); omit anchor with first/last-child to target the root.
3. Finalize (Workflow E).

### Recipe: "update the texts on this page"

This is the request that most often gets mishandled as a full-page replace. It is not one.

1. `gutenberg-get-content` (`target_id`).
2. From the tree, take only the blocks that actually carry the text you were asked to change —
   the ones whose `contentAttribute` holds it (`wd/title`, `wd/paragraph`, `wd/button`, …).
   Ignore every layout wrapper (`wd/section`, `wd/row`, `wd/column`) and every block whose text
   you were not asked to touch. They need no operation at all.
3. Send one `update` op per text block, each carrying **only the content attribute**:

```json
{ "target_id": 123, "expected_content_hash": "…", "operations": [
  { "op": "update", "blockId": "a1b2", "attributes": { "content": "New heading" } },
  { "op": "update", "blockId": "c3d4", "attributes": { "content": "New paragraph copy." } }
] }
```

That is the whole call. No `innerHTML`, no styling attributes, no untouched blocks, no
`block_spec`. Fonts, colors, spacing, images and layout survive untouched precisely *because*
you did not resend them.

## Workflow C — author a block or page from scratch

No example to copy, so discover the schema first:

1. `gutenberg-list-blocks` (optional `category`/`search`) → pick blocks. For a layout, pass the
   `block_category` that `list-layout-types` reports for that layout type.
2. `gutenberg-describe-block` (`block_names`) → per block: `attributes`, `contentAttribute`,
   `example`, `nesting`, `styleGroups`, `composites`, `conventions`.
3. Need styling? `gutenberg-describe-attribute-group` (`groups`).
4. Compose a `block_spec` (nest section → row → column → content), text in attributes.
5. `gutenberg-add-pending-change` (`target_id`, `block_spec`).
6. Finalize (Workflow E).

Page skeleton: `wd/section` › `wd/row` › `wd/column` › content blocks (`wd/title`,
`wd/paragraph`, `wd/button`, `wd/products`, …).

## Workflow D — reuse a library template instead of composing

WoodMart ships a demo template library (the same one the in-editor "Template Library" block
uses). Prefer this over Workflow C whenever the request matches an existing section/page style
(hero, pricing, team, …) — it's faster and visually consistent.

1. `gutenberg-search-templates` (optional `search` title substring, `tag` slug) → pick a
   result's `id`. `available_tags` in the response lists valid tag slugs.
2. `gutenberg-get-template` (`template_id`) → returns `block_spec` (use as-is) and `raw_markup`
   (reference only — do not hand-edit and resend it).
3. Feed `block_spec` into `gutenberg-add-pending-change` (replace a whole target) or as the
   `block` of a `gutenberg-edit-blocks` `insert` op (drop it in at a position). Every `wd/*`
   block in a fetched template already has a fresh `blockId` — no clash, even if you insert
   the same template twice.
4. Finalize (Workflow E). Demo image URLs are localized into the media library automatically
   during finalization — no extra step.

## Workflow E — finalize (REQUIRED, or no block change goes live)

1. Tell the user to open **WoodMart → AI Block Queue** in wp-admin and keep the tab open
   (the browser serializes the blocks with the live `wp.blocks` JS API).
2. `gutenberg-enable-finalization` (`batch_id` from the queue response) — **skip this** when the
   response already says it is marked ready (`create-content` and `create-layout` with a
   `block_spec` do it for you).
3. Poll `gutenberg-get-pending-batch` (`batch_id`) until `status` is `finalized`
   (or `failed` / `conflicted`).
4. `conflicted` = the target changed after you queued → re-read with `gutenberg-get-content`
   and redo the edit.

## Workflow F — theme settings

Settings apply **immediately**; there is no queue and no finalization. The CSS cache is
invalidated for you.

1. `list-theme-settings` (`search` matches setting id **or** label, case-insensitive substring;
   `section` filters by section id; `per_page` default 20). Use it to find the id — do not guess
   slugs.
2. `get-theme-setting` (`id`) → full descriptor, including `default_value`.
3. `update-theme-setting` (`id`, `value`) → returns `{id, old_value, new_value}`.

Read the descriptor before writing. It tells you everything you need:

- **`writable`** — `false` means the AI may not change that field type; the update is rejected.
  Writable types: color, switcher, select, buttons, range, responsive_range, typography,
  text_input, textarea, editor, background.
- **`value_format`** — a `{description, example}` hint for exactly how to shape `value`.
  Highlights: switcher = `"1"` / `"0"`; select/buttons = one of the keys in the descriptor's
  `options`; color = hex string, or an object like `{"idle":"#ff0000","hover":"#cc0000"}` when
  the field is stateful; range = a bounded number; typography/background/responsive_range =
  objects — mirror the shape shown in `current_value` / `default_value`.
- **`has_preset_overrides`** — `true` means a preset also defines this setting.
  `update-theme-setting` writes the **base layer only**, so a preset override can still win and
  the visible value may not change. Tell the user instead of retrying the write.
- **`current_value`**, **`options`**, **`section`**, **`type`**, **`label`**.

## Header logo

`replace-header-logo` (`attachment_id`) swaps the logo image across **every saved header** and
**every logo element inside each one** — a header routinely has two (desktop + mobile), and a site
usually has several headers. One call covers them all. Applies immediately; no queue.

- The logo is not a theme setting and not block content. It lives in each header's structure tree,
  so no other ability can reach it — don't go hunting for a `logo` theme setting.
- Input is a **media-library `attachment_id`** only. Resolve one first; the ability never downloads
  a remote image. A missing or non-image id is an error and nothing is written.
- Headers with no saved data (typically `default_header`, which renders from the theme config) are
  **skipped and reported** with a reason. Relay that: the user must open such a header in the Header
  Builder and save it once to make it editable.
- The **sticky** logo is only replaced where one is already set. An empty sticky logo already reuses
  the main one, so leaving it empty is correct.
- The **footer** logo is a separate theme setting and is not touched here.
- The response reports per-header `status` (`updated` / `skipped` / `unchanged` / `failed`) plus
  `logo_elements_updated`. Read it and relay it — do not report a blanket success.

## Workflow G — layouts

A layout is a `woodmart_layout` template post (single product, shop archive, cart, checkout,
my account, blog, portfolio, product loop item, …). Its blocks are edited with the ordinary
`gutenberg-*` abilities against its `target_id`.

1. `list-layout-types` — always start here. Per type it gives the label, whether WooCommerce is
   required, the `block_category` to pass to `gutenberg-list-blocks`, how many layouts exist,
   and the `predefined` templates available for that type.
2. `list-layouts` (optional `type`) — existing layouts with `id`, `type`, `status`, `conditions`,
   `is_assigned`, `edit_url`, `target_id`.
   - `is_assigned` is resolved in the *current request context*. Over MCP a condition like `all`
     that depends on being on a product/shop page can report `false` even when configured
     correctly. Don't "fix" a layout because of this — check `conditions` instead.
3. `create-layout` (`type`, `title`, optional `status` [`draft`|`publish`, default `draft`],
   `conditions`). Content comes from exactly one of:
   - `predefined_name` (from `list-layout-types.predefined`) → imports a ready template. Content
     is live at once; **no finalization**.
   - `block_spec` → queues authored blocks and auto-marks the batch ready → go to Workflow E,
     step 3 (poll).
   - neither → an empty layout; fill it later with Workflow C/D.
   - `predefined_name` wins over `block_spec`; don't send both.
4. `set-layout-conditions` (`layout_id`, `conditions`) — a **full replacement** set, not a merge.
   `[]` unassigns the layout.

Condition entry shape — `condition_comparison` is always `"include"` or `"exclude"`, plus a
`condition_type` that is valid for that layout type (validation rejects a mismatch). The query
depends on the type:

- **no query** — `all`, `shop_page`, `product_search`, `product_cats`, `product_tags`,
  `product_brands`, `filtered_product_term_any`, `cart`, `empty_cart`, `checkout_form`,
  `checkout_content`, `blog_search_result`, `blog_author`, `blog_date`, `portfolio_search_result`,
  `user_logged_in`, `user_logged_out`.
- **scalar `condition_query`** (required) — the targeted types: `product`, `product_cat`,
  `product_tag`, `product_brand`, `post_id`, `post_cat`, `project_cat`, …
- **array `condition_query`** (required) — `order_shipping_country`, `order_billing_country`.
- **`condition_query_number_min` / `condition_query_number_max`** (at least one required,
  min ≤ max) — `order_total`, `order_subtotal`, `order_subtotal_after_discount`.

```json
[
  { "condition_comparison": "include", "condition_type": "all" },
  { "condition_comparison": "exclude", "condition_type": "product_cat", "condition_query": "hoodies" }
]
```

## Portfolio

Portfolio work splits across two domains, and mixing them up is the usual mistake:

- **One project's content** → it is an ordinary post of type `portfolio`. Find it with
  `search-content` (`post_types: ["portfolio"]`), create one with `create-content`
  (`post_type: "portfolio"`), then read/edit its blocks with `gutenberg-*` against its id,
  exactly like a page. It supports the block editor.
- **How every project or the archive *looks*** → that is a layout, not a project. Use the
  `single_portfolio` and `portfolio_archive` layout types (Workflow G), and target them with
  conditions `project_id` / `project_cat` (single) or `portfolio_category` /
  `portfolio_search_result` (archive).

Both require the Portfolio theme option to be on; without it the `portfolio` post type is not
registered and the content abilities return an explicit error.

## Attribute rules (block authoring)

- **Text/content** → the `contentAttribute` (e.g. `content`). Never hand-write markup.
- **Responsive** (`responsive: true`) → also set `<name>Tablet` / `<name>Mobile`.
- **Units** (`units: "px"`) → companion `<name>Units` holds the CSS unit (default = that value).
- **Style groups** → real name = `prefix` + Capitalized(key), e.g. prefix `tp` + `fontSize` =
  `tpFontSize`. Get keys via `gutenberg-describe-attribute-group`.
- **Composites** (e.g. `advanced`) → shared styling sets; fetch with the group ability.
- **Products / dynamic blocks** → "latest N" needs NO ids: just `orderby` + `items_per_page` +
  `columns`. Specific products/categories need real site ids in `include` / `categoriesIds` —
  resolve them with `woocommerce/products-query`; never invent ids.

## Common mistakes

| Mistake | Do instead |
|---------|------------|
| Sending innerHTML to update text | Forbidden — set only the content attribute; the queue regenerates markup |
| Sending innerHTML "to be safe" alongside attributes | Don't — stale markup can override the new content attribute |
| Forgetting to finalize | Always run Workflow E; poll until `finalized` |
| Calling `enable-finalization` on an already-ready batch | `create-content` / `create-layout` with `block_spec` already did it — just poll |
| **Rebuilding a page to change part of it** (the most common failure) | Rule 1 — `gutenberg-edit-blocks`, only the ops that change. Applies to "update all the texts" too |
| Sending `block_spec` for a page that already has blocks | Only for an empty target or a rebuild the user asked for |
| Resending blocks you were not asked to change | Omit them entirely — untouched blocks are preserved automatically |
| Resending all attributes to change one | `attributes` merges — send only what changes |
| Targeting a core block by blockId | Only `wd/*` blocks have blockIds |
| Inventing product/category ids | Resolve via `woocommerce/products-query`, or use a query (orderby) with no ids |
| Editing after a conflict without re-reading | Re-read `gutenberg-get-content`; pass `expected_content_hash` |
| Guessing a theme-setting slug | Find it with `list-theme-settings` — its search matches the label too |
| Retrying a setting write that "didn't apply" | Check `has_preset_overrides`; updates write the base layer only |
| Merging into `set-layout-conditions` | It **replaces** the whole set; send every condition you want to keep |
| Re-creating a layout just to edit it | Take its `target_id` from `list-layouts` and use `gutenberg-*` |
| Editing a portfolio *layout* when asked to change one project's content | A project is a `portfolio` post — edit it via `search-content` + `gutenberg-*` |
| Queueing/finalizing a title or slug change | `update-content` is immediate; the queue is only for block content |
| Publishing without being asked | `status: "publish"` is outward-facing — confirm first |
| Changing a published page's slug without warning | It 404s the old URL with no redirect — confirm first, then relay `warnings` |
| Trying to delete content | No delete ability exists; tell the user instead of faking it |

## Quick reference

- The target parameter is **always `target_id`** (an integer, required). There is no `post_id`
  alias — the input schemas are `additionalProperties: false`, so sending `post_id` is rejected.
- Any existing post can be a target; no post-type restriction is enforced. In practice that means
  a page, post, `portfolio` project, `cms_block` (all from `search-content`) or a
  `woodmart_layout` (from `list-layouts`).
- One active pending change per target at a time; cancel with `gutenberg-delete-pending-batch`.
- Theme settings and layout creation/conditions are instant. Only **block content** is queued.