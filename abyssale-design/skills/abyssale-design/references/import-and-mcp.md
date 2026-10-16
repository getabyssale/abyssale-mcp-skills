# Import, MCP, and result verification

Read this reference to build/import a design, use an existing reference, or produce renders. It specifies the protocol; composition and Text/Button properties are covered elsewhere in the skill.

Alpha contract: check available tools and their current schemas. Read `version` from `get_design_as_import` rather than assuming one, and check the render for any property the result depends on.

Quick access: [design reads](#discover-tools-and-choose-the-right-read), [fonts](#resolving-fonts), [structure](#assemble-the-multiformat-structure), [import versus generation](#translate-between-import-and-generation), [assets and tracking](#providing-assets-and-tracking-imports), [rendering](#choose-rendering-and-check-all-results), [troubleshooting](#diagnose-common-errors).

## Discover tools and choose the right read

Find the Abyssale tools actually exposed; prefixes depend on the environment. In Codex they may be `mcp__codex_apps__abyssale_<name>`; in Claude the prefix depends on the client (e.g. `mcp__claude_ai_Abyssale__<name>`, possibly deferred: load them with tool search first); short names below denote those capabilities. Read each tool's contract before first use. An older description may be less current than a newly available specialized tool. Do not invent mutation routes absent from the connector.

| Need | Tool and useful information |
|---|---|
| Find a design | `search_designs`, then `list_designs` if needed; keep the returned UUID |
| Identify destination/workspace | `list_projects`, `get_me` when context requires |
| Read type, formats, substitutable names/attributes | `get_design` |
| Read a specific format's values | `get_design_format`, using its returned `id` or `uid` |
| Obtain complete structure for adaptation | `get_design_as_import` |
| Find an actual font | `list_fonts`, filtered by `name`; `type: "custom"` for workspace uploads |
| Learn an official structure | `get_design_import_example`, especially `static-multi-format` |
| Create a design from JSON | `import_design_from_json`, then `check_design_import_status` if needed |

`get_design` starts generation overrides; it is not a complete structural export. `get_design_format` provides flattened state for one format; do not copy it as a multiformat structure.

`get_design_as_import` returns `{payload, warnings, version}`: extract `payload` before reimport. Examine `warnings` before claiming a faithful copy. Export may omit generation settings or internal properties. It does not include the `target` destination; add it explicitly. Reimport creates a new object rather than modifying the source.

Read `printer_multipage` pages through export; it has no formats accessible through `get_design_format`.

Sources: [MCP tools](https://developers.abyssale.com/mcp/tools-reference), [Export](https://developers.abyssale.com/rest-api/designs/as-import), [Two layer contexts](https://developers.abyssale.com/rest-api/layers-overview).

## Resolving fonts

Search the desired family by name instead of browsing the entire catalog. Use the returned `id` in `font`; a name such as `Inter` is not a font UUID. Read `available_weights`: numbers denote upright weights, strings such as `700-italic` denote italic variants. Select an available variant; disclose substitutions and check rendering and line breaks. Do not assume a workspace custom font exists in another workspace. Dry runs do not resolve fonts: valid structure does not guarantee loading.

Visual family selection and draft fidelity are covered in [typographic comparison](native-layers.md#matching-typographic-character); this section concerns technical resolution.

Source: [Fonts](https://developers.abyssale.com/rest-api/fonts) and the current `list_fonts` contract.

## Assemble the multiformat structure

For `static`, `animated`, and `printer`, declare `formats[]` and a shared `layers[]` list. `printer_multipage` uses global print settings and `pages[]`; do not apply this structure to it. Start from the official example for the chosen type and current contract. For print, read units, physical dimensions, DPI, bleed/safe areas, and output settings; do not copy static-example pixel coordinates as print specifications. Validate printer and multipage designs on the real deliverable before delivery.

For a design, provide `target: {"kind":"design","project_uuid":"…"}` with an actually selected UUID. For a workspace template, use `target: {"kind":"workspace_template"}` without a project; it must later become a design before generation. Do not implicitly reuse an example's destination.

Each layer has `name`, `type`, `layout`, and `properties`. Layout uses `x`, `y`, `width`, `height`, top-left origin; integer pixels for static/animated. With multiple formats, **every `layout` and `properties` is keyed by format name**. `default` contains shared values merged underneath format-specific ones. Check each format's effective merged result, particularly all four coordinates and font size. Single-format flat structures do not prove compatibility with a later second format. Text and Button require initial import `font_size`; fitting may change rendered size. Respect type requirements, including after merging `default`.

Educational layer fragment assuming declared `square` and `portrait` formats; not a complete request:

```json
{
  "name": "headline",
  "type": "text",
  "layout": {
    "square": {"x": 70, "y": 100, "width": 1060, "height": 160},
    "portrait": {"x": 55, "y": 100, "width": 790, "height": 190}
  },
  "properties": {
    "default": {"payload": "A clear idea", "color": "#123B36", "font_size": 80},
    "portrait": {"font_size": 72}
  }
}
```

Keys must exactly match declared names; `default` is reserved and cannot name a format. `hidden` and `locked` may remain global booleans or become per-format dictionaries. Layer type is `shape`, geometry is `properties.shape`; group members are in layer-level `layer_ids`.

Preserve unaffected exported layer order. If correcting depth, deliberately adjust the list and verify overlaps in renders; do not invent `z_position` or `z_index`. Do not add `preview_image`: the service produces previews.

Sources: [Create Import](https://developers.abyssale.com/rest-api/designs/import/create), [Shape](https://developers.abyssale.com/rest-api/designs/import/layer-types/shape), [Group](https://developers.abyssale.com/rest-api/designs/import/layer-types/group).

## Translate between import and generation

Never inject an import `properties` object directly into generation `elements`. Names and shapes differ even when the intended effect is similar.

| Intent | Import | Generation, if exposed |
|---|---|---|
| Layer text | `layers[i].properties.…payload` | `elements.<name>.payload` |
| Format background | `formats[i].background_color` | `elements.root.background_color` |
| Text alignment | `text_align` + `vertical_align` | `alignment`, for example `"top left"` |
| Text background padding | `padding: {vertical,horizontal}` | `background_padding: "8 16"` |
| Stroke | `stroke` object | `stroke_width`, `stroke_color` |
| Shadow | `shadow` object | Type-documented `shadow_*` attributes |
| Automatic text shrinking | Initial size and `min_font_size` | `auto_resize` with `min_font_size` |
| Design variables | Text containing variables | `elements.vars`, sibling of layer names |

Use only layer names and `attributes[].id` actually returned by design reads. An unknown field has no effect. Each value under `elements.<name>` must be an object, not a direct string. Preserve existing settings by omitting unchanged fields; omitting `elements` renders saved content. Additional generation limits/options are not automatically valid for import.

Sources: [Element Properties](https://developers.abyssale.com/rest-api/generation/element-properties), [Text](https://developers.abyssale.com/rest-api/generation/element-properties/text), `get_design` and `get_design_format` contracts.

## Providing assets and tracking imports

Choose a transfer feasible in the environment, following the current connector contract:

1. **Abyssale-hosted URL** — preferred whenever available. After a first import, `get_design_as_import` returns each asset as `https://production-banners.s3-eu-west-1.amazonaws.com/templates/v2/…`; generation outputs (`file.url`) live on the same S3. Reusing them needs no upload step, can't be blocked by a third-party site, and the file is copied into the new design. (Any reachable HTTPS URL is fetched in the same call; the advantage here is reliability.)
2. Already hosted HTTPS URL accessible to the service: backend fetches the asset. Brand sites behind a WAF/CDN may refuse Abyssale's fetcher — observed `unreachable_src … HTTP 502` on a brand site, **only at the real import**, not at dry run.
3. Data URI (base64): transferred in the same call, but avoid putting large base64 into context. Keep it to 100 KB at most (base64 is expensive in tokens). Only when a tool reads the file's bytes — never type base64 yourself.
4. Local file (filename): only if you can run a shell on the machine holding the file — the service returns an upload command that you run yourself. Never hand upload commands to the user.

A filename such as `hero.png` does not make the service read a local file. Keep the layer-name/local-path mapping. The connector returns an `upload_command` per layer; replace only the path and preserve field order, with file last. Only one distinct filename is allowed across a layer's formats; use URLs when images differ by format. The same file used in multiple layers needs multiple presigned transfers; a shared URL avoids duplication.

Before necessary recompression/downscaling, assess displayed size; report before/after dimensions and file size without silent quality changes. If the chosen upload route is impossible, prepare a compatible source instead of leaving the user an impractical shell command.

Dry-run with `validate_only: true` when offered, then import the final payload within authorized scope. A dry run does not render or check every external dependency: it does **not** fetch image URLs or resolve fonts. Take illustration names from the bundled lists.

Give hidden Text/Button layers a realistic box, like visible ones.

Read the whole connector response — both `structuredContent` and `content`. If creation is uncertain, read known status or search the project before resubmitting.

`strict: true` makes the import fail when a text can't fit its box. Immediately retain `import_id`. POST is not idempotent: after uncertain timeout, find the result or resume tracking before creating again.

| Returned state | Action |
|---|---|
| `WAITING_FOR_VALIDATION` | Finish requested uploads, then call `check_design_import_status` |
| `QUEUED` / `PROCESSING` | Wait and resume tracking; do not create again |
| `DONE`, finalized | Read `result.uuid`, `result.kind`, and all warnings |
| `FAILED`, finalized | Read the error; fix its cause before a justified retry |

MCP may wait roughly 60 seconds and return the result directly, or return an identifier to track. Use `wait_for_result: false` for obviously long work; avoid tight polling. After uploads, `check_design_import_status` continues the import. Presigned uploads expire: read returned deadlines and do not reuse expired URLs.

Sources: [Import API](https://developers.abyssale.com/api-reference/createDesignImport), [Uploads](https://developers.abyssale.com/rest-api/designs/import/upload-assets), [Status](https://developers.abyssale.com/rest-api/designs/import/status), `import_design_from_json` and `check_design_import_status` contracts.

## No content update: plan imports

The connector can't update a design's layers or formats: every re-import creates a new design. Therefore validate the multi-format layout with a mockup before the first import, iterate with generation overrides (copy, colors, font size, `hidden`, `illustration_file`) that don't change the saved design but prove a fix, use a drafts project for exploration (pick one with `list_projects`, or create one with `create_project` if the user agrees), and name versions (`… (v2)`). When the user approves the final version, list the drafts and test designs and offer to delete them with `delete_design` (needs the `delete:design` scope; no undo — confirm first). The scratch design used for AI images can be kept for later tasks. `rename_design` and `move_design` can rename the final design and move it to the right project (moving resets its review/approval status).

## Choose rendering and check all results

| Need | Tool and routing |
|---|---|
| One format of a `static` design | `generate_static_banner`, format in `template_format_name`; URL returned immediately |
| Multiple static formats, AI generation, animation, or simple print | `generate_banners_async`, list in `template_format_names` |
| `printer_multipage` design | `generate_multipage_pdf`, overrides in `pages.page_1`, `pages.page_2`, etc. |
| Resume asynchronous rendering | `check_generation_status` with `generation_request_id` |

Check design `type`: static png/jpeg/webp/avif/pdf; animated html5/mp4/gif; printer pdf. `generate_banners_async` does not accept `printer_multipage`; its page numbers follow `payload.pages[]` order, starting at 1.

Explicitly select formats for a trial; omitting `template_format_names` requests all. Do not use `original_visual_id` for a new variant: it regenerates an existing visual while keeping its URL. On async completion, inspect both `banners` **and** `errors`; a finalized job can contain partial success. Prefer `file.cdn_url` for presenting renders when available; otherwise use `file.url`, especially for HTML5 ZIP.

Sources: [Synchronous generation](https://developers.abyssale.com/rest-api/generation/synchronous-generation/generate-single-image), [Asynchronous generation](https://developers.abyssale.com/rest-api/generation/asynchronous-generation/), contracts of the three generation tools.

## Separate technical success from visual quality

Technical checks: valid payload, finalized imports, resolved assets, exact names/formats, reviewed warnings, every requested output accounted for. Visual checks: open actual renders and inspect reading, text sizes, framing, margins, overlaps, contrast, rhythm, and cross-format fidelity. `DONE` or zero warnings does not certify graphic quality.

Inspect every delivered adaptation; start with the most demanding aspect ratios to correct early. Successful autofit may make typography too small: fix content or geometry instead of accepting technical success. After correction, target affected formats and risks that actually justify retesting. Keep final payload, identifiers, observed version, relevant warnings, and useful renders; do not present educational fragments as production-validated.

## Diagnose common errors

| Signal | Targeted check |
|---|---|
| `unknown_field` | Wrong import/generation context, nesting level, or another type's property (e.g. `letter_spacing` instead of `char_spacing`, group `padding`) |
| Multiformat `invalid_payload` | Still-flat `layout`/`properties`, malformed color or dependency |
| `unknown_format_key` | Name missing from `formats[]` or case mismatch |
| `missing_required` | Destination, dimension, or mandatory property missing after `default` merge |
| `conditional_dependency_missing` | Missing group reference or mask/filter companion |
| `unknown_font` / `font_resolution_failed` | Unavailable font UUID or variant |
| `text_fit_failed` | Box too small or text too long; read strict/warning and inspect rendering |
| `unreachable_src` / `missing_assets` | Inaccessible URL (a brand site may block Abyssale's fetcher — only at the real import) or unfinished upload → reuse an Abyssale-hosted URL or a data URI; local file upload only if you can run it yourself |
| `cannot fit … min_font_size` at import | Text box too small for its copy (hidden layers included) |
| `cannot_build_banner … cannot fit within the defined space` at render | Copy too long for its box → set `min_font_size` or a larger box |
| `alignment` rejected | Use contract values: `middle left`, not `left center` |
| `property_not_representable` | The export can't carry that property — keep it in your source |
| `feature_not_in_plan` | Feature not covered; an identical retry will not resolve it |

Read `errors[]` and exact returned paths; fix causes instead of blindly removing properties. An override with no effect means a wrong attribute id: reread the design's attributes instead of guessing. Source: [Warnings & Error Codes](https://developers.abyssale.com/rest-api/designs/import/status-warnings-and-errors).
