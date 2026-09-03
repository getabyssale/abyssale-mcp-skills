---
name: abyssale-personalize
description: Generate many personalized creatives from one Abyssale design and a dataset (a CSV, a table, a list of rows) — e.g. one banner per product, per store, per customer name. Use when the user gives or describes a list of rows and wants a creative generated per row from the same base design, such as "generate a banner for each product in this CSV" or "make one variant per city with its own name and price".
---

# Bulk-personalize an Abyssale design from a dataset

This is "mail merge" for creatives: one existing Abyssale design, one row of data per output, many finished creatives without touching the editor. Each row becomes one `generate_banners_async` call with its own element overrides.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` scope (and `read:design`/`read:credits` for the lookup and preflight steps below).

Not for `printer_multipage` designs — those use `abyssale-print-catalog` instead, with a different addressing scheme (pages, not per-row elements).

## Steps

1. **Identify the design.** Use `search_designs`/`list_designs` to find it, `get_design` to confirm it and — critically — to see the real layer names and which attributes are overridable. **Never guess layer names or property names**; an override with a wrong key is either rejected or silently ignored depending on the field, and either way you don't get the personalization you asked for.

2. **Get a concrete reference.** Call `get_design_format` for the format you'll generate, to see one format's current values — useful for knowing what a valid value looks like for each attribute you're about to override (e.g. is a color a hex string, is an image field a URL).

3. **Map the dataset to layers.** Get the user's rows (they may paste a CSV, describe a table, or reference a file) and map each column to a layer name + attribute from step 1. Confirm the mapping with the user before generating anything if any column name doesn't obviously match a layer — a wrong mapping silently produces N wrong creatives, which is expensive to redo.

4. **Preflight the credit cost.** Before firing off a large batch, call `get_remaining_credits` and tell the user how many rows you're about to generate against how many credits are available. Don't launch a 200-row batch that will exhaust the workspace's credits halfway through without saying so first.

5. **Submit one `generate_banners_async` call per row**, with `elements` built from that row's values (layer name → override properties). Pass `template_format_names` if the user wants multiple formats per row, not just one. Submit every row's call before polling any of them — don't wait for row 1 to finish before submitting row 2.

6. **Poll each `generation_request_id`** with `check_generation_status` until every row is finalized. Treat `is_finalized: false` as normal and re-poll later, not as a failure.

7. **Report as one batch**, once everything is done: which rows succeeded, which (if any) failed and why, and the asset URLs — matched back to the row/identifier the user will recognize (product name, city, etc.), not just the raw generation id.

## Notes

- Don't reuse one row's `elements` object as a base and mutate it in place across the loop if you're tracking state carefully — build a fresh override object per row so a bug in one row's mapping can't leak into another's.
- If the dataset is large enough that credits or rate limits are a real risk, say so and offer to run a small batch first (e.g. 3-5 rows) so the user can confirm the mapping looks right before committing to the full run.
