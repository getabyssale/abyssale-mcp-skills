---
name: abyssale-variations
description: Generate outputs from an existing Abyssale design in several of its existing formats, in several languages or text variations, or as a video/GIF/HTML5 banner from an animated design, in one go. Use when the user asks to "generate N variations of this banner", "render this in all its formats", "resize/adapt this banner for LinkedIn", "export this animated design as a video", "make it in French and German", or wants several creative outputs derived from one Abyssale design. For one creative per row of a dataset, see abyssale-personalize. Not for multi-page PDFs from a printer_multipage design — see the abyssale-print-catalog skill for that.
---

# Generate variations from an Abyssale design

Abyssale can turn one design into many rendered outputs without recreating the design each time. What a design can produce depends on its `type`:

| Design type | Outputs (`image_file_type`) |
|---|---|
| `static` | images: `png`, `jpeg`, `webp`, `avif`, `pdf` |
| `animated` | `mp4`, `gif`, `html5` |
| `printer` | `pdf` |

A design's **formats** (sizes) are defined in the design itself — created in the Abyssale editor or by an import. Generation renders those formats; it never creates a new size. Every generation spends Abyssale credits; the amount varies by output, so never estimate it.

Note: `printer_multipage` designs are a different design type and are out of scope here — `generate_static_banner`/`generate_banners_async` both reject them. Use the `abyssale-print-catalog` skill for those.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` and `read:design` scopes.

## Steps

1. **Identify the design.** Use `search_designs` or `list_designs` to find its identifier, then `get_design` to confirm it's the right one and read its `type`, its available formats and its layers.
   - If the user asks for a size the design doesn't have, say so — new formats are added in the Abyssale editor. Don't invent a `template_format_names` value.
   - For a text, language or image change, map it to the real layer names and attributes from `get_design` (call `get_design_format` to see what a valid value looks like). **Never guess layer or property names**: a wrong name is silently ignored (no error, no change). Images must be HTTPS URLs. For a font change, use the `id` from `list_fonts` in `elements.<layer>.font`.

2. **Choose the tool:**
   - **One image, in one format, from a static design, with no AI properties** → `generate_static_banner`. It returns the file directly.
   - **Everything else** → `generate_banners_async`: several formats, animated designs (`mp4`/`gif`/`html5` always go through it), printer designs, and AI image properties (`remove_bg` to remove an image's background, `text_to_image` to generate an image from a prompt — they also spend AI credits). Set `image_file_type` to match the design's type (table above); for a printer design, pass `print.display_crop_marks: true` if the user wants crop marks.

3. **Submit the generation(s).**
   - **One call per content variant** — the design as it is, each language, each text swap — with that variant's `elements` overrides (build a fresh object per call). In each call, list every format wanted for that variant in `template_format_names`; omitting it renders every format of the design. Don't split formats into separate calls.
   - A default call waits up to ~60 seconds and usually returns finished banners directly. For more than a few calls, tell the user how many outputs will be generated, wait for their go-ahead, then submit with `wait_for_result: false`, sequentially or a few at a time, and keep track of which request ID is which variant.
   - **Update a visual in place** only when the user explicitly wants to replace a visual already generated (e.g. "keep the same link"): pass its `id` as `original_visual_id`. The visual is overwritten and keeps its share URL; with `generate_banners_async` the call must produce exactly one format. Confirm first — the previous render is lost.

4. **Collect the results.** Only for requests that came back with `is_finalized: false`, call `check_generation_status`:
   - Still pending → normal, not an error; check again a bit later. If it's still pending after a couple of checks, tell the user which outputs are still rendering.
   - Finalized → account for both `banners` and `errors`; collect URLs only for succeeded outputs. Prefer `cdn_url`; `html5` output is a ZIP with no `cdn_url`, so give its `url`.
   - A generation request can only be looked up for 7 days; after that `check_generation_status` answers `generation_request_gone` — don't retry the same ID.

5. **Report results as a set**, once every requested variation is finalized — which succeeded, which failed and why, and the URLs, labelled by format and variant (e.g. "FR – facebook-post").

## Notes

- Don't fake multi-format generation by looping `generate_static_banner` — one `generate_banners_async` call with several `template_format_names` does it.
- A static design can't be turned into a video, GIF or HTML5 banner — only an animated design can. If the user asks for one from a static design, say so.
