# Runtime: Codex

Read this at the start of every task when Codex drives the skill (ChatGPT web/app without a shell: see [runtime-no-shell.md](runtime-no-shell.md)). Codex has **native image generation and editing (ImageGen)**, a local shell and an image viewer: use them for mockups and raster assets. Do **not** use the scratch-design AI pipeline for ordinary asset creation.

## Tools

Abyssale tools may appear as `mcp__codex_apps__abyssale_<name>`; short names in this skill (`import_design_from_json`, `generate_banners_async`…) refer to them. Read each tool's contract before first use. If the connector is missing, say so; never simulate an import.

## Images: ImageGen first

| Need | Codex path |
|---|---|
| Explore a raster direction (full ad mockup) | ImageGen mockup, then decomposition into native layers ([visual-workflow.md](visual-workflow.md)) |
| Photo, illustration, texture to insert | ImageGen asset at the display ratio; inspect before use |
| Cut-out / extraction of a plane (foreground foliage, subject) | ImageGen edit with transparent output; check RGBA (alpha really 0–255, no fringe), keep a common canvas and coordinates for plates that must align |
| Reconstruct a hidden area / atmospheric texture | ImageGen edit from the reference, removing texts and subjects already in other layers |
| Background removal of a supplied photo | ImageGen edit, or Abyssale `remove_bg` if fidelity to the exact pixels matters and the model result is better — compare |

Follow the available ImageGen skill and built-in tool (if that skill requires the user's confirmation before using CLI/API mode, ask first). Inspect local files before editing and state their role (reference, editing target, asset to insert). Describe the result in the prompt, not the future JSON. A generative cut-out is not a guaranteed pixel-identical extraction: say so for exact products.

**When Abyssale's own AI is still the right choice in Codex:**
- the AI step must run **at render time for every variation** (e.g. `remove_bg` on each product photo of a feed, `auto_focus` crops per image) → put the AI properties on the final design's image layer;
- the image must be regenerable from the design itself (Text to Image on the layer);
- the asset should live on Abyssale's storage without an upload step.
In those cases the recipes in [runtime-claude.md](runtime-claude.md#scratch-design-ai-pipeline) apply (text to image, remove_bg, expand via `generate_banners_async`; async only; AI credits consumed).

## Uploads

The local shell can run the presigned `upload_command` returned for filename sources. Order of preference still applies ([import-and-mcp.md](import-and-mcp.md#providing-assets-and-tracking-imports)): reuse Abyssale-hosted URLs first (no upload, and no risk of the brand site blocking Abyssale's fetcher), then public HTTPS, data URI (100 KB max), local file upload run by you. The Codex sandbox may need network approval for the upload. Brand sites behind a WAF may answer 502 to Abyssale's fetcher only at the real import — upload the file instead.

## Inspecting renders

Download `file.cdn_url` or `file.url` and open it in the image viewer; for MP4/GIF extract frames at key times (ffmpeg). Build a contact sheet for multi-format sets ([capabilities.md](capabilities.md#contact-sheet)).

## Shared rules (also in SKILL.md)

- Use **auto-layout groups** for text stacks; observed order = member layers' order in `layers[]` reversed — not documented, check on the first render ([capabilities.md](capabilities.md#auto-layout-groups)).
- Set **`min_font_size`** on every variable text so longer copy shrinks to fit.
- `validate_only` does not fetch URLs or resolve fonts.
- Illustration layers (feather/material/undraw) with `primary_color` give editable icons and scenes — take Feather/Material names from `references/icons/` (`grep`; classic Material set) and use undraw only with a known name.
- The connector can't edit a design's content: validate the layout with the ImageGen/vector mockup before the first import, iterate with generation overrides, then offer to delete the drafts (`delete_design`, after confirmation).
- Brand tokens from the site's CSS custom properties; regulated-sector mentions on every format.
