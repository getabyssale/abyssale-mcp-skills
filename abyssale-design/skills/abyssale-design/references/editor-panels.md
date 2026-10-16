# Editor-only features

Some options exist in the Abyssale editor but are not part of the import contract. Use this file to tell the user what to finish in the editor after an import, or to guide them step by step when they work in the editor. Menus and model lists change: check the current editor and the [import contract](https://developers.abyssale.com/rest-api/designs/import/layer-types/) before promising an option.

## Set in the editor, not through import

| Feature | Where | Export (`get_design_as_import`) |
|---|---|---|
| Text Max lines, Balance text visually, Truncate text | Text panel (Auto resize section) | Not exported (warning) — keep these settings in your notes and re-apply them after a re-import |
| Text `min_font_size` | Can be set at import and in the editor | Not exported (warning) — keep it in your source payload |
| Custom HTML code layer | Animated designs only (Plus menu) | Not exported (warning) |
| Video masks (Blob, Circle, Hexagon, Parallelogram, Pentagon, Squircle, Rounded Corners) | Video panel | Not exported (warning) |
| Group mask (Rectangle with radius, or Circle) | Group → Add mask to Group | Exported as the group's `mask` |

Read every export warning before reusing an export: a structurally valid export doesn't restore what was dropped.

## Text

- **Partial styles** (weight, size, color, decoration, superscript/subscript, kerning, background) are set by double-clicking the text on the canvas and selecting characters. In JSON they are markup inside one Text layer (see [native-layers.md](native-layers.md)). A partial size entered in px is stored relative to the base size (e.g. 32 px on a 40 px base → `<f=80%>`); a partial background's padding and radius are stored on the layer (`padding`, `background_radius`).
- **Background** modes: Highlight, Paragraph, Fill entire width. Fill entire width removes horizontal padding and X offset.
- **Side Border** (one side of the text: thickness, padding, rounded/straight, offset) is different from the glyph **Border**.
- [Auto Resize](https://help.abyssale.com/en/articles/92216-mastering-text-auto-resize-layout-harmony), [Full-width background](https://help.abyssale.com/en/articles/89590-apply-a-full-width-text-background), [Side Border](https://help.abyssale.com/en/articles/546336-using-the-side-border-decoration-on-text-layers).

## Button

Two separate shadows: **Button Shadow** (the box) and **Text Shadow** (the label). Box alignment, label alignment and padding: see [native-layers.md](native-layers.md). [Button documentation](https://help.abyssale.com/en/articles/89592-adding-a-button-layer).

## Image and logo

- **Update image** sources: Computer, URL, Unsplash, Pexels, Pixabay. **Update logo**: Computer, URL, Brand Presets.
- **AI options** on an image layer: Remove Background, Auto focus (Generic or People), Text to Image, Expand. Model lists change — for the API, use the [Image generation properties](https://developers.abyssale.com/rest-api/generation/element-properties/image), and don't translate editor labels into API ids.

## Group

Auto Layout offers a direction, a gap and a nine-cell alignment grid (static designs only — no Auto Layout on animated groups). [Auto Layout](https://help.abyssale.com/en/articles/233796-group-layers-auto-layout-in-abyssale-design-builder), [Masked Groups](https://help.abyssale.com/en/articles/620021-using-masked-groups-in-the-abyssale-design-builder).

## Animation

| Layer type | Animations offered |
|---|---|
| Text | Fade, Slide, Scale, Rotate, Typewriting (Classic, Soft fade, Focus in, Rise up, Rise down) |
| Button, Shape, Logo, Image, QR Code, Rating, Illustration | Fade, Slide, Scale, Rotate |
| Video | Fade, Slide, Scale |
| Audio | Audio Fade (in/out) |

Easing is available on every visual animation: None, Back.InOut, Bounce.InOut, Circ.InOut, Elastic.InOut, Power2.In, Power2.Out, Sine.InOut, plus the workspace's own Bézier curves. To add a keyframe, place the timeline cursor on a free time inside the tween first.

## Illustration & icons

Libraries: Feather, Undraw, Material, Twemoji. Feather, Undraw and Material can be recolored; Twemoji keeps its own colors.
