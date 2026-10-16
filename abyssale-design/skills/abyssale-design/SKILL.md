---
name: "abyssale-design"
description: "Create, reproduce, adapt or fix editable designs in Abyssale through the Abyssale MCP connector — static, animated, print or multipage — from a brief, a brand URL, a reference or an existing design, including AI images and cut-outs. Works in Claude, ChatGPT and Codex. Use whenever the user wants to create or change an Abyssale design, template, banner, ad set or format variations; not for simply finding a design or rendering one unchanged."
---

# Design and produce in Abyssale

Preserve artistic intent and intended use through an **editable, native structure** that survives new copy, new formats and hand edits in the Abyssale editor. No style, industry, number of formats, subject type or advertising composition is prescribed; the product and the channel set the technical limits.

## 0. Identify the runtime first

This skill is shared by several agents. The Abyssale contract is the same for all of them; what differs is how images are produced, how files move and how renders are inspected.

| You are running in… | Read first |
|---|---|
| **Codex** (native ImageGen, local shell and image viewer) | [runtime-codex.md](references/runtime-codex.md) |
| **Claude** with a shell — Cowork, Claude Code, claude.ai with code execution (no native raster generation) | [runtime-claude.md](references/runtime-claude.md) |
| **No shell** — ChatGPT web/app, claude.ai without code execution, other chat clients | [runtime-no-shell.md](references/runtime-no-shell.md) |
| Anything else | Check the tools actually present (shell, web access, image viewing, image generation), then use the closest runtime file |

Rule of thumb: **Codex makes images with ImageGen; Claude makes images with Abyssale's own AI through a scratch design** (text to image, remove background, expand); without a shell, images come from URLs, Abyssale's AI or native illustrations. Steps that need a shell (downloading, scripts, local previews) apply only when one is available — the no-shell file gives the alternative for each. Everything else below is common.

## Read only the relevant references

| Need | Reference |
|---|---|
| Runtime-specific tools, image generation, uploads, render inspection | runtime-codex.md / runtime-claude.md / runtime-no-shell.md |
| Verified native catalog: shapes, patterns, masks, filters, gradients, illustrations, text/button props, image AI, format presets, auto-layout | [Capabilities](references/capabilities.md) |
| Creative direction, fidelity, depth, assets, adaptation by format | [Design and decomposition](references/visual-workflow.md) |
| Layer types, typography, rotation, Button, markup, SVG lettering | [Layers and text](references/native-layers.md) |
| JSON, tools, fonts, uploads, tracking, troubleshooting | [Import and MCP](references/import-and-mcp.md) |
| Rating, Illustration, Video, Audio, animation | [Specialized layers](references/specialized-layers.md) |
| Editor-only features | [Editor panels](references/editor-panels.md) |
| Small working payloads | [Examples](references/examples.md) |

Where a behavior isn't proven, the references say so ("check the render", "check after importing"). The import contract is Alpha: read tool schemas before first use, read `version` from `get_design_as_import` instead of assuming one, and re-check what the result depends on.

## 1. Choose the workflow

Identify objective, exact content, references, assets, formats, medium and expected editability. Decide ordinary choices yourself; ask only what materially changes the result. Never invent offers, figures, performance numbers, data or official assets.

| Request | Starting point |
|---|---|
| Open-ended creation | Brand research → direction → mockup (runtime image path, vector mockup, or a first native import) |
| Reproducing a reference | Treat it as the target; do not regenerate what must be preserved |
| Targeted correction | Read the design + full export; change only the cause |
| Format adaptation | Preserve invariants and recompose for the new formats |
| Dynamic template | Separate fixed/variable content; plan long/short copy, languages, image swaps |

Establish the output type: static, animated, print or multipage. Typographic, editorial or abstract visuals need no photo, cut-out or CTA. Before a structural change to an existing design, read its formats and its full export (take `.payload` from `get_design_as_import` and add a `target`). If the fix can be made in the editor, guide the user through it rather than re-importing. A re-import creates a new object; a generation override never modifies the saved design.

## 2. Define the visual target

**Brand research from a URL.** Official CSS is the most reliable source of tokens: download the page and extract CSS custom properties (`--color-*`, `--font-*`), dominant hex colors, logo SVG and font files. Check logo fills against the planned background (a navy wordmark vanishes on navy), open candidate images, copy official taglines verbatim, note mandatory legal mentions. Look for the brand font among workspace fonts (`list_fonts(name=…, type="custom")`) before substituting a Google font; disclose substitutions.

**Regulated sectors** (finance, health, alcohol, gambling, credit…): mandatory risk mention on every format, non-promissory benefit wording, only figures the brand publishes, compliance validation flagged to the user.

Define hierarchy, composition, palette, typographic intent and rhythm. Keep creative freedom at mockup stage (no font or specimen imposed unless the brief requires it). From the chosen direction, list visible invariants — text silhouette and weight, colors and punctuation, size ratios, angles, motifs, placements, front/back relations. A clean but weaker rebuild is not a faithful one. When feedback is "too plain", change a real lever (composition, title mass, illustration, contrast, iconography), not random ornaments.

## 3. Derive the structure

Run the [depth and overlap analysis](references/visual-workflow.md#depth-and-overlap-analysis) when elements intersect. Choose each editable unit by behavior:

- Format background for flat, linear- or radial-gradient fills — not an Image layer.
- Text for editable copy, with partial styles via markup in one layer.
- Button for label + background that change together; Shape for independent geometry (capsule = rectangle `radius: 999`, exact circle = `circle`).
- **Illustration layers** (feather, material_icons, undraw — recolorable with `primary_color`) for icons and flat scenes that must stay editable. Take names from `references/icons/*.txt` — with a shell, search them with `grep` (never read the lists whole); otherwise search the file's text. Use undraw only with a known name.
- Image for photos, textures, complex art, custom SVG; Logo for supplied logos.
- **Group with `auto_layout`** for anything that must stack or align (headline → subline → bullets → CTA; icon + label rows).

Highly stylized lettering without a close font may become outlined SVG in an Image layer; keep other text native and disclose the loss of text editing. Record decisions (stable layer names, role, type, z-order, per-format geometry, variations) in the build script, or in your notes without a shell.

## 4. Adapt formats and content

Recompose each ratio: reading axes, crops, line breaks, relative sizes, motifs, content zones. Use the format presets and safe zones in [capabilities.md](references/capabilities.md#format-presets). On small formats hide optional elements before shrinking everything; never drop mandatory mentions.

### Safe zones constrain content, never the decor (full bleed)

Every format has two geometries: the **canvas** (edge to edge) and the **safe area** (inside margins and platform-UI zones, e.g. story top ~250 px / bottom ~340 px). They govern different things:

- **Safe area → important content only**: headline, offer, CTA, logo, legal mentions, key face or product. These stay inside it.
- **Canvas → everything that is decor**: photo, illustration, scene, sky, ground, texture, color fields, decorative shapes. These **always continue to all four edges**, including through the unsafe zones.

Consequences:

- **Never leave dead space outside the safe area.** No flat empty band, no plain-color margin, no letterbox left behind because content was pulled inward. If a zone outside the safe area holds nothing important, the scene continues there (the image keeps going, the illustration's sky/floor/background extends, the pattern runs off the edge).
- **Decor image boxes cover the whole canvas** (they may overflow it). When the source image does not cover the new ratio — a square scene in a 9:16 story, a portrait photo in a 1.91:1 landscape — *extend the image*: outpaint it (`expand` through the scratch pipeline / ImageGen), generate it at the target ratio, or recompose with a wider/taller source. Never shrink it and fill the rest with background color.
- **Illustrations and vector scenes**: draw the scene to the full format (one SVG per ratio, or an oversized canvas with the key subject in the safe center) so background elements bleed off every edge.
- **Text panels / color blocks** are allowed as deliberate composition, but they must reach the canvas edges they touch *and* be filled by content or decor. If part of a panel ends up empty (typically the bottom of a story below the legal mention), let the image continue behind instead: shorten the panel to its content, or replace it with an overlay/gradient over the continuing image (Image `overlay`, or a gradient Shape) that keeps the text legible.
- Cut-outs, pop-outs and overlaps are welcome in the unsafe zones (a subject can cross into them); text and CTAs are not.

Two rules that prevent most layout defects:
- **Auto-layout stacks** instead of hand-placed blocks: the vertical order follows the member layers' order in `layers[]`, **reversed** (last in `layers[]` = top of the stack); `layer_ids` order has no effect (observed, not documented — check on the first render).
- **`min_font_size` on every variable text**, so longer copy shrinks to fit its box.

For animation and print, apply the medium-specific checks; a preview PNG is not enough (print: decor extends into the bleed too). For display formats, offer an animated HTML5 variant when a simple motion idea exists.

## 5. Build and save

With a shell, generate the payload with a script (template in [capabilities.md](references/capabilities.md#payload-builder-template)); without one, start from `get_design_import_example` or `get_design_as_import` and always run `validate_only` first ([runtime-no-shell.md](references/runtime-no-shell.md)). Follow the [MCP protocol](references/import-and-mcp.md). Fragile points:

- Resolve real font UUIDs and variants; add an explicit `target`.
- Multi-format: key every `layout` and `properties` by format name; `default` only for shared values. Give hidden Text/Button layers a realistic box.
- Import properties ≠ generation attributes ≠ UI controls.
- `validate_only` checks structure only — it does **not** fetch image URLs or resolve fonts.
- Assets, in order: Abyssale-hosted URLs (reuse those returned by `get_design_as_import` or by a generation — no upload, and no risk of a site blocking Abyssale's fetcher) → public HTTPS (brand sites behind a WAF may return 502 to Abyssale) → data URI (base64, 100 KB max, read from the file by a tool — never typed) → local file upload, only if you can run the returned `upload_command` yourself. Never hand upload commands to the user.
- Keep the `import_id`; imports are not idempotent. Read all warnings.
- AI steps cost AI credits that vary by model: never estimate a cost in advance. Before more than a few AI calls, say how many you plan; afterwards report the credits actually used (`get_remaining_credits` before and after).
- **The connector can't edit a design's content.** Every re-import creates a new design: validate the layout before the first import (a local mockup of every format is cheap), iterate cheaply with generation overrides, and name versions clearly. Once the final version is approved, offer to delete the drafts with `delete_design` — after the user confirms, as there is no undo — and to rename (`rename_design`) or move (`move_design`) the final design. The scratch design used for AI images can be kept for later tasks.

Editor-only features: see [editor panels](references/editor-panels.md). After any edit in the editor, save, then re-read the design.

## 6. Verify and deliver

Render the **saved** design (one `generate_banners_async` for several formats, or `generate_static_banner` per format) and inspect every format as an image — a contact sheet helps. Check exact copy, graphic impact, real rendered font sizes, holes left by wrapping, margins and safe zones, crops, contrast of key objects, legal mentions, overlaps, CTA box vs label. **Check the edges**: along each border (especially story top/bottom and landscape sides), the decor must continue — any flat empty band or unused panel area outside the safe zone is a defect to fix, not a margin. `DONE` and zero warnings prove nothing visually. Test a long-copy variant on the most constrained format. Fix causes, re-render affected formats, then stop iterating.

Deliver: link/id and project, a real preview, formats produced, substitutions (fonts, assets), non-editable parts (SVG art, vector lettering), which images are AI-generated and AI credits used, drafts and test designs to delete (the scratch design can be kept), items needing brand or compliance validation. Keep the payload, SVGs and assets as files. Distinguish prepared / imported / saved / rendered / inspected. Remove construction leftovers hidden everywhere; in targeted fixes, only clean what your change made useless.