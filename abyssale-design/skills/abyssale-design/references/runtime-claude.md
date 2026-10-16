# Runtime: Claude (Cowork, Claude Code, claude.ai)

Read this at the start of every task when Claude drives the skill. It covers what differs from other runtimes: tool discovery, image production without native raster generation, file transfers and render inspection.

## Tools and environment

Abyssale tools come from the connector (`https://mcp.abyssale.com/mcp`), with a client-specific prefix (e.g. `mcp__claude_ai_Abyssale__<name>`). They are often **deferred**: load them with tool search (`abyssale import`, `abyssale fonts`, `abyssale generate`) before calling. If none appears, say the connector is not enabled for this conversation — never simulate an import.

| Capability | Cowork / Claude Code | claude.ai chat |
|---|---|---|
| Shell + Python sandbox | yes | yes, outbound network allow-listed |
| Presigned S3 upload (`upload_command`) | works | blocked unless a workspace admin allowlists `https://production-templates-imports.s3.eu-west-1.amazonaws.com` (see the MCP overview docs) — try once, then fall back |
| Download renders (`file.url`, S3 host) and view them with the image Read tool | works | often blocked (`cdn.abyssale.com` observed blocked); ask for a screenshot, never claim an inspection you did not do |
| Local SVG/HTML preview | `pip install cairosvg`, or Playwright with the pre-installed Chromium | same if the package registry is reachable |
| Editor UI | only with a browser-control tool (Claude in Chrome / built-in browser) | guide the user step by step, then re-read the design |

Practical cautions:
- Never print a base64 data URI to the console: the harness may try to parse it as an image. Print its length instead.
- Generate large payloads with a script and paste the printed JSON (`ensure_ascii=True`) into the tool call; never hand-type them.
- Deliver files (payload JSON, SVGs, overview image) through the file-delivery mechanism of the surface (outputs folder, file attachment, connected folder).

## Producing images without native raster generation

Claude cannot generate raster images itself. Use, in order:

1. Brand or user-supplied assets (inspect them first).
2. **Abyssale's own AI through a scratch design** (below) — photos, cut-outs, extended backgrounds, AI illustrations.
3. Native illustration layers (undraw, feather, material icons — recolorable, stay editable).
4. Vector SVG drawn by Claude (flat illustrations, shapes, lettering).
5. An image-generation connector actually present in the session.

Never present an SVG drawing as a photograph, and say which images are AI-generated.

### Scratch-design AI pipeline

Abyssale's image AI runs at generation time on an image layer. To produce standalone assets, use a dedicated scratch design whose only job is to run it; its outputs then populate the final design.

**1. Reuse or create the scratch design.** `search_designs` for `SCRATCH ai-assets` in the project; otherwise import once (no upload needed) — full payload in [examples/scratch-ai-design.json](examples/scratch-ai-design.json): one Image layer `img` filling formats `sq` 1024×1024, `wide` 1600×900, `tall` 900×1600, all with background `#FFFFFF00`. The transparent background plus `image_file_type: "png"` makes cut-outs come back with a real alpha channel.

**2. Run one AI step per call** with `generate_banners_async(design_id, template_format_names=["sq"], image_file_type="png", elements={"img": {…}})` (text to image is rejected by the synchronous tool). The output is `banners[0].file.url` on Abyssale's S3.

| Goal | `elements.img` |
|---|---|
| Text to image | `{"text_to_image": true, "text_to_image_properties": {"prompt": "…", "model": "nano-banana-2", "ratio": "1:1"}}` |
| Remove background | `{"image_url": "<url>", "remove_bg": true, "remove_bg_properties": {"model": "bria-rmbg-2-0"}}` |
| Generate + cut out | both properties in the same element (works in one call) |
| Extend to another ratio | on `wide`/`tall`: `{"image_url": "<url>", "fitting_type": "fill", "expand": true, "expand_properties": {"model": "flux-2-pro-outpaint"}}` — ignored with `cover` |
| Crop on subject (check the result) | `{"image_url": "<url>", "auto_focus": true, "auto_focus_properties": {"model": "people", "focus_framing": "shoulders"}}` |

**3. Inspect** each output: download it, composite cut-outs on the target background color, check alpha, halos and leftovers. Iterate on the prompt, not the layout.

**4. Post-process locally if useful** (Pillow): trim to alpha bbox (`im.crop(im.split()[3].getbbox())`), pad, resize to display size. Bring a processed file into the final import following the upload order below — usually a data URI (100 KB max) or a local file upload you run yourself.

**Full-bleed adaptation of one scene to several ratios**: generate the master once (e.g. `tall` 9:16), then run a single `expand` call with `template_format_names=["sq","wide"]` and `fitting_type: "fill"` — each format returns its outpainted version. Cut out the subject on *each* expanded image (expand may slightly redraw it), then place photo and cut-out with identical boxes whose aspect equals the source image (boxes may overflow the canvas) so they stay pixel-aligned. A scratch design with bigger formats (sq 1536², wide 2048×1152, tall 1152×2048) gives sharper outputs than the 1024 defaults.

**5. Use the output URL directly as `src` in the final import** — Abyssale fetches its own storage without fetch problems and copies the file into the design storage.

Pitfalls:
- With `remove_bg_crop: true`, the trimmed image is re-fitted according to `fitting_type`: use `fill` to keep the whole subject, or leave it off and trim locally.
- Prompts: describe the final use (subject, framing leaving room for copy, light, brand hex palette, "no text, no logo, no watermark"); for later cut-outs ask for "isolated on a plain pure white background"; match `ratio` to the scratch format; avoid low-contrast accents.
- Expand re-synthesizes surroundings and may slightly redraw the subject — never on an exact product shot.
- Never generate real identifiable people, logos or products presented as real, or fake testimonials. AI people are illustrative models; regulated brands may forbid AI imagery — ask.
- Each AI call spends AI credits plus a generation credit; amounts vary by model and can change, so never estimate a cost. Before more than a few AI calls, tell the user how many you plan; afterwards report the credits actually used (`get_remaining_credits` before and after).
- AI imagery in ads may need disclosure (EU AI Act Art. 50, ad-platform policies) — flag it.
- Put AI directly on the final design's image layer only when each variation needs its own AI step at render time (e.g. removing the background of every product photo in a feed) — it then costs AI credits on every render.
- Keep the scratch design for later tasks, or offer to delete it (`delete_design`, after the user confirms).

### Vector illustrations drawn by Claude

A flat, brand-colored SVG scene generated with a short Python script works well as an Image layer:
- square `viewBox` (e.g. 1000×1000), transparent background, brand colors + 1–2 accents;
- tell the headline's story visually (e.g. a headline about growth → a plant growing out of a coin stack);
- preview with `cairosvg.svg2png(bytestring=svg, background_color=<format bg>)` and check contrast of key objects against what is behind them;
- square layout boxes with `fitting_type: "cover"` so the art never crops;
- tell the user one SVG = one layer whose parts are not individually editable; prefer native layers for anything that changes per campaign.

## Uploads

Order of preference (details in [import-and-mcp.md](import-and-mcp.md#providing-assets-and-tracking-imports)): Abyssale-hosted URL → public HTTPS → data URI ≤100 KB → filename + `upload_command`. From Cowork/Claude Code, run each `upload_command` yourself, substituting only the local path, then poll `check_design_import_status`. In claude.ai, try once; if the proxy refuses, fall back to a hosted URL or a recompressed data URI (report before/after size) — do not hand the user a curl command.

## Inspecting renders

Download `file.url` and read the PNG; build a contact sheet (script in [capabilities.md](capabilities.md#contact-sheet)) and look at every format. If downloads are blocked, share the links, describe what was checked technically (status, warnings, `get_design_format` values) and ask for visual feedback.
