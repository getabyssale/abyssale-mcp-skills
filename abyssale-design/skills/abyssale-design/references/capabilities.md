# Capabilities catalog and verified recipes

Applies to every runtime (Claude, Codex, no-shell clients).

## Format presets

| Name | Size | Notes |
|---|---|---|
| square | 1080×1080 | feed |
| portrait | 1080×1350 | feed 4:5 |
| story | 1080×1920 | keep text/CTA out of roughly the top 250 px and bottom 340 px (platform UI) — the decor (photo, illustration, tints) still runs through those zones to the edges |
| landscape | 1200×628 | link ads / LinkedIn / FB |
| mrec | 300×250 | IAB; hide secondary copy |
| half-page | 300×600 | IAB |
| leaderboard | 728×90 | IAB; headline + CTA + logo only |
| skyscraper / mobile leaderboard | 160×600; 320×50 | extreme ratios: simplify hard |

Recompose each ratio (axes, crops, sizes, alignments); don't just scale. On small formats hide *optional* elements (eyebrow, subline, icon rows) — never mandatory mentions. Guidance, not a platform rule: keep legal text ≥ ~8 px on IAB formats and ≥ ~16 px on social formats.

## Auto-layout groups

Hand-placing headline → subline → bullets → CTA by estimating line wraps leaves holes whenever a title wraps on fewer lines than expected. Use a vertical auto-layout group:

```json
{"name": "stack", "type": "group", "layer_ids": ["headline", "subline", "cta"],
 "layout": {"square": {"x": 64, "y": 150, "width": 560, "height": 400}},
 "properties": {"default": {"auto_layout": true, "direction": "vertical", "gap": 24, "alignment": "top left"}}}
```

Behavior:
- **Stacking order follows the order of the member layers in `layers[]`, reversed** (the layer that comes *last* in `layers[]` — the top-most in z-order — is placed *first*). `layer_ids` order has no effect. Observed, not documented — check on the first render. So to read headline → subline → CTA from top to bottom, declare them in `layers[]` as `cta`, `subline`, `headline`.
- Member boxes hug their content; the group `alignment` positions the whole stack inside the group box (`middle left` centers it vertically).
- Use `direction: "horizontal"` for icon + label rows (one small group per row). Nested groups inside an auto-layout group aren't covered here — verify them on a render.
- Not allowed on groups: `padding`, `opacity`, `rotation`, `background_color` (→ `unknown_field`); auto-layout is rejected on **animated** designs.

### Variable text must be allowed to shrink
A text that doesn't fit its box at its `font_size` can only shrink down to `min_font_size`. Set `min_font_size` (import) on every variable text so longer copy shrinks to fit, then test a long variant. `min_font_size` isn't exported by `get_design_as_import` — keep it in your source.

## Native catalog

**Gradients** — Abyssale syntax, not CSS:
- Linear: `"linear-gradient(0% 0% 0% 100%,0% #EFE6DF 1,100% #F8D4D2 1)"` — 4 coordinates (x1 y1 x2 y2: `0% 0% 0% 100%` = top→bottom, `0% 0% 100% 0%` = left→right).
- Radial: `"radial-gradient(50% 50% 50%,0% #EFE6DF 1,100% #F8D4D2 1)"` — center x, center y, radius (> 0).
- **2 to 8 stops**, each `offset% color opacity` (offset 0–100 %, opacity 0–1). All stop colors hex (`#RRGGBB`/`#RGB`) or all `cmyk(…)`, not mixed. Transparency goes in the stop's opacity number, not as alpha in the stop color.
- Where: static/animated → format background, Shape/Button `background_color`, Text `color`. Printer designs → Shape/Button `background_color` only, with opaque stops.

**Transparent tints over a full-bleed photo**: fractional alpha stops (`0% #25368B 0.85,100% #25368B 0`) and 8-digit hex on a shape (`#25368BD9`) both import and render. Pattern: solid tint shape behind the copy column + gradient shape fading into the photo; the scene stays visible to every edge while text stays legible. Put a cut-out of the subject above the tints so it stays vivid.

**Shapes** — `properties.shape`: rectangle, triangle, right_triangle, circle, half_circle, star_1, star_2, pentagon, hexagon, hexagram, rhombus, parallelogram, trapeze, squircle, curved_diamond, window, frame, arrow, cross, gear, rounded_gear, blob, cloud, heart. Props: `background_color`, `pattern_name` + `pattern_color`, `opacity`, `rotation`, `stroke {color,width}`, `radius` (int or `{tl,tr,bl,br}`), `shadow {blur,offset_x,offset_y,color}`.
**Patterns** (13): `bubbles, cage, cross, doubleCircle, drops, parkay, pills, plus, star, ticTac, ticTacFilled, triangle, wiggle`. Good for texture blocks without any image.

**Image** (import): `src`, `fitting_type` cover|fill, `alignment` (9-grid or custom) + `position_offset_x/y` (−0.5..0.5), `zoom` 1–20, `opacity`, `rotation`, `mask_name` (circle, rounded_corners+`mask_properties.radius`, blob, squircle, pentagon, hexagon, parallelogram, window), `filter_name` grayscale|duotone + `filter_properties.name` (blue_green, blue_orange, deep_green_light_green, BrightRed_light_yellow, brown_pale_green, brown_beige, deep_blue_green, deep_blue_red, deep_purple_orange, deep_purple_beige, deep_purple_pale_green, deep_blue_pale_green), `overlay {direction,color_1,color_2}`, `shadow`, `remove_bg` (+`remove_bg_properties`) at import.

**Image AI at generation time** (recipes in runtime-claude.md, scratch-design pipeline; costs AI credits): `text_to_image` / `text_to_image_properties {prompt (≥3 words), model (default nano-banana-2; current list and ratios: [Image generation properties](https://developers.abyssale.com/rest-api/generation/element-properties/image)), ratio, quality, inpaint_images}` — **async only** (sync returns 400). `expand` (outpainting; ignored with cover; async only). `remove_bg` (models bria-rmbg-2-0, birefnet, pixelcut, imageUtils, ideogram; async recommended). `auto_focus` (generic/people, framing, zoom, target; sync OK). Use these via `generate_banners_async` — on a scratch design to produce assets (Claude), or ImageGen for one-off assets (Codex), or directly on the final design's image layer when the AI step must re-run for every variation.

**Illustration layers**:
```json
{"type": "illustration", "properties": {"illustration_type": "feather", "illustration_file": "trending-up",
 "illustration_properties": {"primary_color": "#FD7B3D"}, "alignment": "middle center"}}
```
- Libraries: `feather` (names from feathericons.com, e.g. `layers`, `key`, `trending-up`), `material_icons` (snake_case ids from fonts.google.com/icons, e.g. `visibility_off`), `undraw` (file names without extension, e.g. `building_blocks`, `city_life`), `twemoji` (fixed colors).
- `primary_color` recolors the accent of undraw scenes (e.g. bricks) and the whole feather/material icon.
- **Take names from the bundled lists — search them, never read them whole:**
  - `grep -ix "home" references/icons/feather.txt` — 287 Feather names (feather-icons 4.29.2).
  - `grep -i "estate" references/icons/material-icons.txt` — 2,234 **classic** Material Icons names (Google `MaterialIcons-Regular.codepoints`). Abyssale's `material_icons` library is the classic Material Icons set (filled style), not Material Symbols.
  - `undraw`: no list is bundled — use an undraw name only if the user supplies it or it already works in an existing design; otherwise prefer Feather or Material.
- Icons in a navy circle: Shape `circle` + Illustration at ~50 % of the circle size, centered.

**Text** (import): `payload`, `font` (UUID from `list_fonts`), `font_weight`, `font_style`, `font_size` (required), `min_font_size`, `line_height` (% of size, default 130), `char_spacing` (−100..1000; *not* `letter_spacing`), `text_align`, `vertical_align`, `text_transform`, `background_mode` highlight|paragraph, `background_fill` normal|width, `padding`, `side_border`, `rotation` (0–360, e.g. 355 for −5°). Not accepted on Text: `letter_spacing`, `max_lines`, `text_shadow`.

**Button**: `payload`, font props, `color`, `background_color`, `padding {vertical,horizontal}`, `radius`, `alignment` (places the visible box inside the layout), `text_align` (lines inside), `stroke` (border of the button), `shadow` (box) vs `text_shadow` (glyphs). Always set an explicit `padding`: the visible box is sized from the label plus padding.

**Import vs generation names differ**: `padding` → `background_padding` ("8 16"); `text_align`+`vertical_align` → `alignment`; `stroke` → `stroke_width`/`stroke_color`; format background → `elements.root.background_color`. Use only attribute ids returned by `get_design`: an unknown key has no effect.

## Payload builder template

Multi-format rule: every layer's `layout` and `properties` are **keyed by format name**; `default` holds shared values merged under each format. Flat shape is rejected as soon as there are 2 formats. Text/Button need `font_size` after merge. Give hidden Text/Button layers a realistic box, like visible ones.

Template:

```python
import json
F = {"square": (1080,1080), "portrait": (1080,1350), "story": (1080,1920), "landscape": (1200,628),
     "mrec": (300,250), "half-page": (300,600), "leaderboard": (728,90)}
def b(x,y,w,h): return {"x":x,"y":y,"width":w,"height":h}
G = {  # per-format geometry and sizes in ONE place
  "square": dict(logo=b(64,60,200,78), head=(b(64,172,580,230),62), cta=(b(64,764,420,84),26)),
  # ...
}
HIDE = {"subline": {"mrec","leaderboard"}}
def keyed(fn): return {f: fn(f) for f in F}
def text(name, key, font, weight, color, payload, **extra):
    L = {"name": name, "type": "text", "layout": keyed(lambda f: G[f][key][0]),
         "properties": {"default": {"payload": payload, "font": font, "font_weight": weight,
                                    "color": color, **extra},
                        **{f: {"font_size": G[f][key][1]} for f in F}}}
    if name in HIDE: L["hidden"] = {f: f in HIDE[name] for f in F}
    return L
layers = [...]   # z-order: first = bottom. Remember the auto-layout ordering rule (above).
payload = {"target": {"kind": "design", "project_uuid": PROJECT}, "name": NAME, "type": "static",
           "formats": [{"name": f, "width": w, "height": h, "background_color": BG} for f,(w,h) in F.items()],
           "layers": layers}
json.dump(payload, open("payload.json","w"), ensure_ascii=False)
print(json.dumps(payload, ensure_ascii=True))   # paste this exact output into the tool call
```

Never print a base64 data URI to the console (it can be mis-parsed as an image by the harness); check its length instead.

## Contact sheet

```python
from PIL import Image
ims = [Image.open(p).convert("RGB") for p in paths]; H = 700
row = [i.resize((int(i.width*H/i.height), H)) for i in ims]
sheet = Image.new("RGB", (sum(i.width for i in row) + 20*len(row), H), "white"); x = 0
for i in row: sheet.paste(i, (x, 0)); x += i.width + 20
sheet.save("overview.jpg", quality=88)
```

## Error quick reference

See [Diagnose common errors](import-and-mcp.md#diagnose-common-errors).
