---
name: abyssale-personalize
description: Generate many personalized creatives from one Abyssale design and a dataset (a CSV, a table, a list of rows) — e.g. one banner per product, per store, per customer name. Use when the user gives or describes a list of rows and wants a creative generated per row from the same base design, such as "generate a banner for each product in this CSV" or "make one variant per city with its own name and price".
---

# Bulk-personalize an Abyssale design from a dataset

This is "mail merge" for creatives: one existing Abyssale design, one row of data per output, many finished creatives without touching the editor. Each row becomes one `generate_banners_async` call with its own element overrides. Every generation spends Abyssale credits; the amount varies by output, so never estimate it.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` and `read:design` scopes, plus `read:font` to look up fonts with `list_fonts`.

Not for `printer_multipage` designs — those use `abyssale-print-catalog` instead, with a different addressing scheme (pages, not per-row elements).

## Steps

1. **Identify the design.** Use `search_designs`/`list_designs` to find it, `get_design` to confirm it and — critically — to see its formats, the real layer names and which attributes are overridable. **Never guess layer names or property names**: a wrong layer or property name has no effect, and an override must be an object (e.g. `{"payload": "…"}`, not a bare string).

2. **Get a concrete reference.** Call `get_design_format` for the format you'll generate, to see one format's current values — useful for knowing what a valid value looks like for each attribute you're about to override (e.g. is a color a hex string). Image values must be public HTTPS URLs; if the dataset has local files instead, ask the user for links. For a font, use the `id` returned by `list_fonts` in `elements.<layer>.font`.

3. **Map the dataset to layers.** Get the user's rows (they may paste a CSV, describe a table, or reference a file) and map each column to a layer name + attribute from step 1. Confirm the mapping with the user before generating anything if any column name doesn't obviously match a layer — a wrong mapping produces N wrong creatives, which is expensive to redo.

4. **Submit one `generate_banners_async` call per row**, with `elements` built from that row's values (layer name → override properties). Always pass `template_format_names` with exactly the formats wanted — omitting it renders every format of the design for every row. Before a large batch, tell the user how many rows × formats will be generated and wait for their go-ahead. Then set `wait_for_result: false`, submit sequentially or a few at a time, and keep the mapping from each row to its `generation_request_id`. If a response already came back finalized, use its `banners` and `errors` directly.

5. **Poll each pending `generation_request_id`** with `check_generation_status` until every row is finalized. Treat `is_finalized: false` as normal and re-poll later, not as a failure. A finalized result has both `banners` and `errors` — account for both. A request can only be looked up for 7 days (`generation_request_gone` after that).

6. **Report as one batch**, once everything is done: which rows succeeded, which (if any) failed and why, and the asset URLs (prefer `cdn_url`) — matched back to the row/identifier the user will recognize (product name, city, etc.), not just the raw generation id.

## Notes

- Build a fresh `elements` object for each row instead of reusing and mutating one across the loop, so a value from one row (e.g. when the next row's cell is blank) never carries over into the next.
- For a large dataset, offer to run a small batch first (e.g. 3-5 rows) so the user can confirm the mapping looks right before committing to the full run.
