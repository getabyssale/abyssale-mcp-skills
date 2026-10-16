---
name: abyssale-print-catalog
description: Generate a multi-page PDF (catalog, brochure, lookbook, multi-page menu) from an Abyssale multi-page print design (type printer_multipage). Use when the user asks to "generate the PDF catalog", "export this brochure as a PDF", "make a multi-page print piece", or "generate one catalog per store". Not for single-page flyers or posters, single banners, social formats, or video — see abyssale-variations for those.
---

# Generate a multi-page PDF catalog from Abyssale

`printer_multipage` is a distinct Abyssale design type: a fixed sequence of print pages, not the `formats[]` a regular banner design has. Generation fills the existing pages — it never adds pages. If the content needs more pages than the design has (e.g. 40 products for 10 pages), tell the user: pages are added in the Abyssale editor. It needs its own generation call and its own way of addressing content: `generate_static_banner` and `generate_banners_async` both reject this design type outright. Every generation spends Abyssale credits; the amount varies, so never estimate it.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` and `read:design` scopes.

## Steps

1. **Confirm the design is `printer_multipage`.** Use `search_designs`/`list_designs` to find it, then check its type before proceeding — if it isn't `printer_multipage` (e.g. a single-page `printer` flyer), this is the wrong skill; use `abyssale-variations` instead.

2. **Get the page structure before writing any overrides.** Call `get_design_as_import` on the design (`get_design` doesn't show the layers of each page). It returns every page's `payload.pages[]` entry, in order, with each page's `layers[]`. **Never guess page, layer or property names**:
   - Pages have no id/name field of their own; you address the Nth page positionally as `page_1`, `page_2`, … (1-indexed, matching array order).
   - Layer names come from each page's `layers[].name`.
   - Property names are the keys under each layer's `properties` (e.g. `payload` for a text). Copy the format of the existing values — print colors may be `cmyk(…)`.

3. **Build the overrides.** For each page you want to change, pass an object keyed by that page's layer names (from step 2), each mapping to the override properties for that layer — `{ "page_N": { "<layer name>": { …overrides } } }`. `pages` is required: pass `{}` to export the catalog with its saved content unchanged. Images must be HTTPS URLs.

4. **Call `generate_multipage_pdf`** with `design_id` and the `pages` object from step 3. Two important behaviors:
   - **Wrong page/layer/property names fail silently** — an unknown key is accepted and simply does nothing, no error. If a page didn't change the way you expected, re-check the names against `get_design_as_import`'s output rather than assuming the API rejected something.
   - It waits internally for up to ~60 seconds and returns the finished PDF directly if ready in time. Pass `wait_for_result: false` up front if the catalog has many pages (likely slow) or the user said not to wait — you'll get a `generation_request_id` back immediately instead.

5. **If you got back a `generation_request_id`** (either because `wait_for_result: false` was set, or the 60s window ran out), poll it with `check_generation_status` — the same status tool used for async banner/video generation. Once finalized, give the user the PDF URL from `banners` and surface any `errors`. A request can only be looked up for 7 days (`generation_request_gone` after that).

6. **Print options**: pass `print.display_crop_marks: true` if the user wants crop marks. Don't set `print.color_profile` unless the user gives you a color profile UUID (a name like "CMYK" or "FOGRA39" can't be passed). Otherwise leave `print` unset.

7. **One PDF per row** (e.g. one catalog per store from a dataset): one `generate_multipage_pdf` call per row with a fresh `pages` object each time. Before the batch, tell the user how many PDFs will be generated and wait for their go-ahead; offer a test on 1–2 rows first; submit with `wait_for_result: false` and keep track of which request is which row.

A multi-page PDF can't be regenerated in place — each call produces a new PDF.
