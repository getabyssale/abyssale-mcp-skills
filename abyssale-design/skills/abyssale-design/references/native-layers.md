# Native layers, Button, and partial styles

The contract is Alpha — read `version` from `get_design_as_import`. Settings not available through the MCP are listed in [MCP limits](mcp-limits.md); small working payloads are in [examples](examples.md). Composition choices are recommendations, not API constraints.

## Import vs generation

| Feature | JSON import | MCP generation |
|---|---|---|
| Structure, dimensions, formats | `formats/layers/layout/properties` | Creates neither layers nor formats |
| Global background color | `formats[].background_color` | `elements.root.background_color` |
| Partially styled text | Markup in `payload` | Markup in `elements.name.payload` |
| Weight/italics | Numeric `font_weight` + global `font_style`; local italics via markup | Check exposed attributes and markup; do not invent generation `font_style` |
| Text background/padding | Object/number `padding`, background options | String `background_padding`; separate contract |
| Text fitting | Initial `font_size` required, optional `min_font_size` | Documented options; verify availability on the design |
| Button box and label | `alignment` + `text_align` | Do not assume interchangeability; read current attributes |
| Button radius | Scalar `radius` | No documented radius override |
| Stroke/shadow | `stroke`, `shadow` objects | Flattened fields such as `stroke_width`, `shadow_blur` |

Sources: [Layers overview](https://developers.abyssale.com/rest-api/layers-overview), [Generation properties](https://developers.abyssale.com/rest-api/generation/element-properties), [Text import](https://developers.abyssale.com/rest-api/designs/import/layer-types/text), [Text generation](https://developers.abyssale.com/rest-api/generation/element-properties/text), [Button import](https://developers.abyssale.com/rest-api/designs/import/layer-types/button), [Button generation](https://developers.abyssale.com/rest-api/generation/element-properties/button).

Quick access: [layer type](#choosing-the-right-type), [typographic fidelity](#matching-typographic-character), [rotation](#block-rotation-and-italics), [text color](#text-color), [Button](#button-beyond-ctas), [markup](#multiple-styles-in-one-text-layer), [SVG lettering](#highly-stylized-svg-lettering), [Text constraints](#text-with-backgrounds-and-constraints).

## Choosing the right type

- **Format background:** a supported native solid fill or gradient without an extra layer. Import: `formats[].background_color`; generation override: `elements.root.background_color`. For `static` / `animated`, linear and radial gradients use [Abyssale syntax](https://developers.abyssale.com/rest-api/designs/import/colors), not CSS (see [capabilities](capabilities.md)); on printer designs only a Shape/Button background can hold a gradient. An effect outside the contract may justify a separate asset when fidelity requires it.
- **Text:** continuous phrases, headlines, paragraphs, legal copy. Use native backgrounds before adding a shape if suitable.
- **Button:** a text/background unit that should change together. Its name does not require a CTA role.
- **Shape:** independent background, exact circle, motif, or native geometry. Geometry uses `properties.shape`, not `properties.type`. Per-corner radius belongs to types that support it, not Button.
- **Capsule behind a logo:** `rectangle` Shape, `background_color`, and sufficiently large `radius` for semicircular ends; separate logo above. Adapt radius to height or use an accepted large value, then check rendering. Preserve color, order, and dimensions during correction; neither SVG nor an empty Button is needed for this geometry.
- **Custom shape beyond native capabilities:** path-based SVG in a separate Image layer, transparent outside the shape. Overlay it on the photograph and set visibility per format; follow [photo/shape/subject decomposition](visual-workflow.md#decorative-shapes-independent-of-photography).
- **Image:** photography, texture, complex illustration, cutout asset. Import accepts `fitting_type: "cover"|"fill"` only (no `contain`). Consult filters and masks before use.
- **Logo:** supplied brand asset with simple framing. Use Image for masks or advanced fitting; do not invent equivalent Logo properties.
- **Group:** independent objects that should move/be organized together. `layer_ids` references names at layer level. It is not a card with its own background/padding.
- **Rating, QR Code, Illustration, Video, Audio:** prefer specialized types when relevant and read their references. For Rating, the four Illustration families, Video, Audio, and animation, see [specialized contracts](specialized-layers.md) and [MCP limits](mcp-limits.md). Video/audio relate to animated designs.

Sources: [Shape](https://developers.abyssale.com/rest-api/designs/import/layer-types/shape), [Image](https://developers.abyssale.com/rest-api/designs/import/layer-types/image), [Logo](https://developers.abyssale.com/rest-api/designs/import/layer-types/logo), [Group](https://developers.abyssale.com/rest-api/designs/import/layer-types/group), [Available types](https://developers.abyssale.com/rest-api/designs/import/layer-types/).

### Specialized layers

The [specialized matrix](specialized-layers.md) covers library choices, Rating limits, Video/Audio contracts, and animation effects. The import contract has **11 layer types**. A **Custom HTML code** layer (animated designs) isn't importable or exported; don't confuse it with `qrcode`.

## Alignment values

Use contractual `alignment` values: vertical first, then horizontal — `middle left`, not `left center`.

These properties can vary by format. Reserve `properties.default` for truly shared values and override differences. Example Button import fragment: `properties: {default: {alignment: "middle left", text_align: "center"}, skyscraper: {alignment: "middle center", text_align: "center"}}`. The label remains centered in the button, while the visible box moves from the left edge to the center of its layout. For Text, instead use `properties.skyscraper.text_align: "center"` and choose `vertical_align` separately.

### Matching typographic character

After choosing the direction, identify family or silhouette, letter width, stroke contrast, cap height, apparent weight, spacing, and angle. Compare relevant candidates using the actual words at comparable visual sizes; a similar family name or identical weight number does not guarantee the same appearance.

Resolve families and variants with `list_fonts` using the [protocol](import-and-mcp.md#resolving-fonts). Set family and variant before compensating with size, box width, line spacing, or kerning. Check the actual autofit size: a requested large headline may have been reduced. Distinguish italics, block rotation, and an angle built into an SVG.

Compare the rendered headline, subtitle, and secondary information according to each role; do not apply one weight to artificially uniformize the design. For exceptional lettering without a close candidate, use the [SVG exception](#highly-stylized-svg-lettering). For variable text, prefer a font compatible with expected content. This research happens during reconstruction and does not impose a font or specimen on raster exploration.

### Block rotation and italics

Identify the reference text block's angle separately from glyph slant. `font_style: "italic"` selects a typographic variant; `skew_y` distorts text; `rotation` rotates the layer. Do not substitute one for another to reproduce an angled headline.

**Documented for Text import:** `rotation` is numeric degrees from 0 through 360, in `properties`, not `layout`. Use 355 for −5°. Multiformat fragment: `properties: {square: {rotation: 355}, skyscraper: {rotation: 0}}`. Merge with other layer properties. Source: [Text import](https://developers.abyssale.com/rest-api/designs/import/layer-types/text). Confirm other types' contracts before generalizing.

After rotation, check visible bounds, margins, collisions, and clipping in every format. Adapt angle, placement, and dimensions together without automatically flattening the title horizontally. Account for SVG lettering already angled in its paths to avoid applying rotation twice.

### Text color

A plain payload uses the layer's `color` for the current format. To color part of a text, use markup (`<c=#FFFFFF>…</c>`); to change the whole text in one format, set that format's `color`. When formats need different colors, set them per format.

## Button beyond CTAs

The visible box is sized from the label plus `padding` and placed inside the `layout` box by `alignment`; `text_align` aligns the label lines. Always set an explicit `padding`. `stroke` on a Button draws the box border.

| Recommended use | Properties | Benefit | Limitation / alternative |
|---|---|---|---|
| "New" badge | Colors, weight, padding, large radius | Text and background move/change together | Icon, ribbon, or star needs extra components |
| Category label | Subtle radius, padding, box alignment | Size follows the label | Fixed-size background: Shape + Text |
| Elongated pill | High radius, short text, padding | Compact component | Exact circle: circle Shape + Text |
| Short callout or box | Multiline, line spacing, left text alignment, padding, border/shadow | One editable unit | Columns, icons, large fixed panel: multiple layers |
| Fixed-width banner | Consider Text + Paragraph/width background | Native typographic background | Background shared by content: separate Shape |

The benefit is consistent editing, not absolute layer minimization. Keep separate objects when behaviors differ.

Badge configuration (import properties fragment; full context in [native-components-multiformat.json](examples/native-components-multiformat.json)):

```json
{
  "payload": "NEW",
  "font": "<FONT_ID>",
  "font_size": 30,
  "font_weight": 700,
  "color": "#FFFFFF",
  "background_color": "#007C70",
  "padding": {"vertical": 16, "horizontal": 28},
  "radius": 48,
  "alignment": "middle left",
  "text_align": "center"
}
```

Resolve the font and the destination project in the target workspace.

**Import ≠ generation:** `padding` becomes `background_padding` in documented overrides; radius cannot be changed through generation. Import `shadow` targets the box, `text_shadow` the glyphs. A setting in Button Text does not prove generation support.

Additional source: [Adding a Button Layer](https://help.abyssale.com/en/articles/89592-adding-a-button-layer).

## Multiple styles in one text layer

Partial styles are markup tags inside one Text layer's `payload` (import and generation). [Help Center — markup](https://help.abyssale.com/en/articles/344945-using-markup-language-on-text-layers-in-abyssale).

### Exact representation

| Effect | Payload |
|---|---|
| Local color | `<c=#007C70>word</c>` |
| Local weight | `<w=700>word</w>` |
| Local italics | `<w=400-italic>word</w>` — use a variant returned by `list_fonts` |
| Underline | `<u>word</u>` or `<u=#152A35>word</u>` |
| Strikethrough | `<s>word</s>` or `<s=#152A35>word</s>` |
| Diagonal strikethrough | `<d>word</d>` or `<d=#152A35>word</d>` |
| Highlight | `<bg=#FFE4A3>word</bg>` |
| Relative size | `<f=70%>word</f>` |
| Absolute size | `<f=110>word</f>` |
| Superscript/subscript | `m<sup>2</sup>`, `H<sub>2</sub>O` |
| Kerning | `<k=-20%>.</k>` |

Sources: [Text generation](https://developers.abyssale.com/rest-api/generation/element-properties/text), [Kerning](https://help.abyssale.com/en/articles/304286-advanced-text-formatting-customize-kerning-in-abyssale), [Markup](https://help.abyssale.com/en/articles/344945-using-markup-language-on-text-layers-in-abyssale).

Do not invent `<span>`, `<b>`, `<i>`, or CSS. Do not extrapolate all italic variants: use a variant actually returned by `list_fonts` and verify rendering. For global style, import separately documents `font_style: "italic"`; it differs from local markup.

Tags can combine; preserve closing tags and nesting. `\n` in a JSON string represents a newline. Local decorations override global ones. Preserve exact rich payload when replacing content; starting from plain text erases emphasis.

### Fragmented versus native structure

For a phrase like "One word in color in the sentence," separate `phrase_1`, `red_word`, and `phrase_2` layers require spacing adjustments after every text change. Prefer one layer:

```json
{"payload": "One word <c=#D3502C><w=700>in color</w></c> in the sentence"}
```

This is a fragment, not a complete design payload. Similarly, a simple background/label capsule can become a Button. Conversely, an independently rotated word or moving icon justifies separate layers.

### Limitations and fallbacks

- Local sizes may disable auto resize (Help Center).
- A partial background's color is in `bg`; its padding and radius are stored on the layer (`padding`, `background_radius`). Don't invent local `bg` parameters.
- For settings not available through the MCP ([list](mcp-limits.md)), say so; if a compatible native structure exists, use it and explain the tradeoff.
- Split for independent placement, distinct animations, or a demonstrated limitation. Group when useful; do not turn all text into an image.

## Highly stylized SVG lettering

When the lettering is highly stylized and no available font comes close after comparison, prefer **SVG lettering imported into an Image layer** over a substitution that weakens the creative direction. This exception applies to the affected headline or words; keep subtitles, CTAs, legal copy, and ordinary text native. It does not impose a font or specimen before the raster mockup.

Reproduce the selected lettering's contours, silhouette, thickness, spacing, colors, and punctuation. Export letters as vector paths in a standalone transparent SVG with an appropriate `viewBox`. Do not merely use SVG `<text>` dependent on an unavailable font, or wrap a bitmap in SVG and call it vectorized. Check edges and readability before import; do not promise pixel identity from a raster draft.

Import through the actual available image mechanism, then check Abyssale rendering, colors, transparency, and clipping in every format. Preserve proportions; create lettering layout variants if another ratio requires different line breaks instead of stretching the title. Verify actual support through rendering, not this recommendation.

Keep the source SVG and exact transcription in deliverables. Explain that lettering remains movable and resizable as an image, but its words are no longer editable through Abyssale's text field. If the brief explicitly requires dynamic or fully editable text, resolve this tradeoff with the user before freezing it. Do not keep an everywhere-hidden Text duplicate solely to archive the copy.

## Text with backgrounds and constraints

For a banner, compare Button with Text + `background_mode: "paragraph"` and `background_fill: "width"`. With `background_fill: "width"`, horizontal padding and X offset don't apply; vertical padding and radius remain. [Full-width background](https://help.abyssale.com/en/articles/89590-apply-a-full-width-text-background).

Max lines, Balance text visually and Truncate text aren't available through the MCP, and `min_font_size` isn't exported (see [MCP limits](mcp-limits.md)). Keep `min_font_size` in your source payload, re-apply it when you re-import, and test longer text. [Auto Resize](https://help.abyssale.com/en/articles/92216-mastering-text-auto-resize-layout-harmony), [Export](https://developers.abyssale.com/rest-api/designs/as-import).

Text side borders use an import `side_border` object and flattened generation properties. Do not confuse them with Button borders. [Side Border](https://help.abyssale.com/en/articles/546336-using-the-side-border-decoration-on-text-layers).
