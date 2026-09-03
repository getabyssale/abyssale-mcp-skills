---
name: abyssale-print-catalog
description: Generate a print-ready multi-page PDF (catalog, brochure, lookbook, menu) from an Abyssale printer_multipage design. Use when the user asks to "generate a PDF catalog", "export this brochure as a PDF", "make a multi-page print piece", or names a design that's a printer_multipage type. Not for single banners, social formats, or video — see abyssale-variations for those.
---

# Generate a print-ready PDF catalog from Abyssale

`printer_multipage` is a distinct Abyssale design type — built as a sequence of print pages (with bleed/safe area/dpi), not the `formats[]` a regular banner design has. It needs its own generation call and its own way of addressing content: `generate_static_banner` and `generate_banners_async` both reject this design type outright.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` scope (and `read:design` for the lookup step).

## Steps

1. **Confirm the design is `printer_multipage`.** Use `search_designs`/`list_designs` to find it, then check its type before proceeding — if it isn't `printer_multipage`, this is the wrong skill; use `abyssale-variations` instead.

2. **Get the page structure before writing any overrides.** Call `get_design_as_import` on the design. This returns every page's `payload.pages[]` entry, in order, with each page's `layers[].name`. **Never guess page or layer names** — pages have no id/name field of their own; you address the Nth page positionally as `page_1`, `page_2`, … (1-indexed, matching array order).

3. **Build the overrides.** For each page you want to change, pass an object keyed by that page's layer names (from step 2), each mapping to the override properties for that layer — same override shape as `generate_banners_async` uses per-element.

4. **Call `generate_multipage_pdf`** with `design_id` and the `pages` object from step 3. Two important behaviors:
   - **Wrong page/layer names fail silently** — an unknown key is accepted and simply does nothing, no error. If a page didn't change the way you expected, re-check the names against `get_design_as_import`'s output rather than assuming the API rejected something.
   - It waits internally for up to ~60 seconds and returns the finished PDF directly if ready in time. Pass `wait_for_result: false` up front if the catalog has many pages (likely slow) or the user said not to wait — you'll get a `generation_request_id` back immediately instead.

5. **If you got back a `generation_request_id`** (either because `wait_for_result: false` was set, or the 60s window ran out), poll it with `check_generation_status` — the same status tool used for async banner/video generation.

6. **Print options**: pass `print.display_crop_marks` if the user wants crop marks on the exported PDF, and `print.color_profile` if they specify one — otherwise leave `print` unset and let Abyssale use the design's own defaults.

## Notes

- Don't try to reach a `printer_multipage` design through `generate_static_banner`/`generate_banners_async` — they reject it outright, this isn't a workaround-able limitation.
- Don't invent page or layer names — always source them from a fresh `get_design_as_import` call, since a silent no-op is much harder to debug than an upfront lookup.
