---
name: figma-to-abyssale
description: Recreate a Figma design (Design file or FigJam board) as a native, multi-layer, fully-editable Abyssale design via the Abyssale MCP server. Use whenever the user gives a Figma URL and wants it rebuilt in Abyssale (import_design_from_json), not just referenced.
---

# Figma → Abyssale conversion

Goal: take a Figma node, extract its structure/styles via the Figma MCP, and rebuild it
as native Abyssale layers (text/shape/image), decomposed element-by-element rather than
flattened into one screenshot — unless the source itself is already a single flattened
image (see "When NOT to decompose" below).

## 0. Tools available (exactly 2, no more)

This uses the community "Framelink" MCP (`figma-developer-mcp`, GitHub `GLips/Figma-Context-MCP`).
Verified by reading its source (`src/services/figma.ts`): it is a **pure passthrough** to the
official Figma REST API (`https://api.figma.com/v1`) — no extra logic, no extra endpoints.

- `mcp__figma__get_figma_data(fileKey, nodeId?, depth?)` → calls `GET /files/:key` or
  `GET /files/:key/nodes?ids=...`. Returns a simplified but **style-complete** tree:
  fills/colors (resolved through a `GLOBAL_VARS` lookup table), strokes, effects, opacity,
  borderRadius, typography (fontFamily/fontWeight/fontSize/lineHeight/letterSpacing/align),
  auto-layout (mode/gap/padding/sizing), and the `NODES` tree (children in z-order),
  plus `COMPONENTS`/`COMPONENT_SETS` for instances.
- `mcp__figma__download_figma_images(fileKey, nodes, localPath, pngScale)` → calls
  `GET /images/:key?ids=...`. Downloads REAL image files (png/svg) locally.
  `imageRef` (from a node's `fills[].imageRef`) is required for image-fill rectangles;
  omit it for vector/SVG nodes (auto-detected). `pngScale` controls real resolution.

Install (if not already connected): `claude mcp add figma -e FIGMA_API_KEY=<token> -- npx -y figma-developer-mcp --stdio`.
Token needs the **"File content: Read"** scope (Figma → Settings → Personal access tokens).
Wrong scope (e.g. only `file_dev_resources`) → `403 Invalid scope(s)... requires file_content:read`.
A newly-added MCP server needs a session restart before its tools show up via ToolSearch.

## 1. Supported Figma file types (verified empirically this session)

| Type | URL pattern | Status |
|---|---|---|
| Design file | `/design/...` | ✅ supported, tested |
| FigJam board | `/board/...` | ✅ supported, tested (2 full designs built) |
| Figma Buzz | `/buzz/...` | ❌ confirmed unsupported — Figma's own REST API returns `400 "File type not supported by this endpoint"` on `GET /files/:key`. Not an MCP bug (verified in source: no file-type filtering happens in the MCP, the 400 comes straight from api.figma.com). No workaround exists through this API. |
| Figma Slides | `.deck` | ⚠️ likely unsupported — per Figma community forum, Slides nodes only exist in the proprietary Kiwi binary format (`fig-deck`), not exposed via REST API. Not personally tested. |
| Figma Sites / Make | — | not tested, no info found |

**If a URL has no `node-id`**, it points at the file root — call `get_figma_data(fileKey, depth: 2)`
first to list top-level frames/pages, then re-call with the right `nodeId`.

## 2. Rate limits (starter plan, Viewer/Collaborator seat — hit repeatedly this session)

Error looks like:
```
Error fetching file: Figma API rate limit hit (429). Retry after <seconds>. Your Figma seat type
(Viewer or Collaborator) has a lower API rate limit. Your starter plan has limited API access.
```
**Verified exact number from Figma's own rate-limits doc** (`developers.figma.com/docs/rest-api/rate-limits/`):
on a Starter plan with a Viewer/Collaborator seat, **Tier 1 endpoints are capped at 20 requests per
MONTH**. Tier 1 = exactly the endpoints this pipeline depends on: `GET file`, `GET file/nodes`,
`GET image` — i.e. every `get_figma_data` and `download_figma_images` call. This is a **monthly
quota**, not a recurring short-term rate limit — don't describe a retry delay of several days as
"the rate limit is 4-6 days," that's just how far into the current monthly cycle you happen to be
before it resets. Correction logged 2026-09-01 after stating this imprecisely to the user.

Practical consequence: **20 Tier-1 calls/month is very easy to exhaust** — a single moderately
complex design rebuild can use 5-10 calls (one `get_figma_data` + several `download_figma_images`
for each asset/reference render). Budget calls accordingly: batch multiple nodes into one
`download_figma_images` call when possible (it accepts an array), and avoid extra reference-image
downloads once the layout is already understood from a prior render in the same file.

No dashboard in the Figma web UI shows live quota usage (verified via docs) — the only UI pointer
Figma gives is an `X-Figma-Upgrade-Link` response header pointing to `/pricing` or `/settings` when
the limit is hit. If this happens: report the exact error + retry delay to the user, do not
loop/retry, and ask whether to wait, upgrade the plan/seat, or use a different token.

## 3. Reading `get_figma_data` output

- `GLOBAL_VARS`: a dedup table of shared `layout_*`, `fill_*`, `style_*` values referenced by
  key from nodes below (saves tokens on repeated styles). Resolve these before mapping.
- `NODES`: the actual tree, indented, in z-order. Node line format:
  `[TYPE] "name" #id layout={...} fills=... strokes=... textStyle=... text="..."`.
  Types seen: FRAME, TEXT, RECTANGLE, ELLIPSE, INSTANCE, IMAGE-SVG, SHAPE_WITH_TEXT, CONNECTOR, GROUP, SLOT.
- Position is **relative to the parent** (`locationRelativeToParent: {x, y}` or implicit in
  auto-layout). To get absolute canvas position for the Abyssale payload, sum offsets down
  the tree (recursive walk from the root frame, accumulating x/y).
- **Hug-sized elements** (`sizing: {horizontal: hug, vertical: hug}` or no `dimensions` at all)
  have NO explicit width/height in the data — you must compute it yourself (child bbox + padding,
  or from a `download_figma_images` pixel size ÷ pngScale). **Known bug from a past session**:
  forgetting this for 2 nodes (`bubble_blue`, `name_pill`) silently dropped them from the rebuild
  because a hand-written tree-walker skipped emitting layers when w/h was `None`. Never leave a
  hug-sized element's box uncomputed.
- Colors come pre-resolved to hex/rgba in `fills`/`strokes` (via `GLOBAL_VARS`) — no manual
  Figma-variable resolution needed on your end.

## 4. Mapping Figma nodes → Abyssale layer types

| Figma | Abyssale |
|---|---|
| `TEXT` | `text` layer — `payload` = the text, `color`, `font` (see §5), `font_weight`, `font_size`, `line_height` (%, Figma `lineHeight: 1.4em` → `140`), `text_align`, `vertical_align` |
| `RECTANGLE`/`ELLIPSE` (no image fill) | `shape` — `shape` (`"rectangle"` by default; an `ELLIPSE` is `"circle"`), `background_color`, `radius` (from `borderRadius`), `stroke: {color, width}` (nested, not flat), optional `shadow` |
| `RECTANGLE` with `fills[].type == "IMAGE"` | `image` — download via `download_figma_images` using that fill's `imageRef`, `fitting_type: "cover"` (Abyssale only accepts `cover`/`fill`, never `contain`) |
| `IMAGE-SVG` | `image` — download via `download_figma_images` (no `imageRef` needed, vector auto-detected) |
| `RECTANGLE`/`ELLIPSE`/`VECTOR` used as an **organic/irregular vector shape** (blobs, wavy decorative lines, dot patterns, thin accent strokes) | `image`, same as `IMAGE-SVG` — see §6b, this is not a native `shape` even though the Figma type looks like RECTANGLE/ELLIPSE |
| `SHAPE_WITH_TEXT` (e.g. sticky notes) | split into 2 layers: a `shape` (bg/radius) + a `text` (centered, using the node's `name` or its text content) |
| `CONNECTOR` | approximate as a thin `shape` (2-4px wide rect) along the bounding box — exact bezier/elbow routing is not worth reproducing natively; disclosed simplification, consistently used across sessions |
| Complex `INSTANCE` (toolbars, keycaps, deeply nested UI chrome) | **judgment call**: either fully decompose (preferred, per "no grouping" default) or flatten to one downloaded image if genuinely decorative chrome not worth decomposing — **always disclose this compromise explicitly to the user**, don't decide silently |
| Pure transparent layout `FRAME` with no fill/stroke of its own | skip — it's just an auto-layout container, not a visible layer |

## 5. Colors and fonts

- Color: Figma hex/rgba → Abyssale hex — `#RGB`, `#RGBA`, `#RRGGBB` or `#RRGGBBAA` (8-char with
  alpha). No 10-char hex, no `rgb()`/`rgba()` strings. If alpha is already embedded, don't append an
  extra `FF`. Gradients have a native string form — see §7.
- Font: Abyssale needs a font **UUID** (the `id` returned by `list_fonts`), not a family name — call `list_fonts` to find the closest
  match (Figma project fonts like "Kaftan Serif - Trial" or "Nanum Pen" are almost never
  available in Abyssale — substitute the nearest available font and mention the substitution).
- `line_height`: Abyssale takes a percentage; Figma gives `em` (e.g. `1.4em` → `140`, `1em` → `100`).

## 6. When NOT to decompose

If `get_figma_data` on the target node returns literally a single `RECTANGLE`/node with one
image fill and no children, the source design was already delivered as one flattened image —
there is nothing to decompose. Import it as a single `image` layer at native pixel size
(get exact dimensions via `download_figma_images`). This is not a violation of "no grouping" —
that rule is about not artificially flattening *multiple distinct* elements, not about
inventing layers that don't exist in the source.

## 6b. Organic/vector shapes → always `image`, never native `shape`

Abyssale's native `shape` layer only supports a fixed list of **primitives** set by its `shape`
property (`rectangle`, `circle`, `triangle`, `star_1`/`star_2`, `pentagon`, `hexagon`, `rhombus`,
`arrow`, `heart`, `cloud`, `blob`… — use one when the Figma node really is that primitive) — it cannot
reproduce an arbitrary bezier path. So any Figma element that is
visually an irregular vector — organic blobs, decorative wavy lines, a column of small dots grouped
as one vector, a thin diagonal accent line — must be downloaded as a real image
(`download_figma_images`, no `imageRef` needed for pure vectors) and placed as its own `image` layer,
sized/positioned from the Figma node's own bounding box.

**This is still full decomposition, not grouping** — each irregular shape stays its own distinct,
individually-positioned layer; only *its internal representation* is a raster image instead of a
native primitive. Only collapse multiple *different* vector elements into one image if they were
already a single Figma node to begin with.

Validated end-to-end on a real design (`Test Yass` — teal/orange poster with 2 organic blobs, a
wavy line, a dot-pattern column, and a thin diagonal accent line, all reproduced as separate
`image` layers at their native Figma bounding box, alongside 3 native `shape`/photo layers and
6 native `text` layers — 14 layers total, visual match confirmed against the Figma render).

**Circular photo crops**: an `ELLIPSE` node with an `IMAGE` fill (a Figma "photo in a circle" pattern)
maps to one `image` layer with `mask_name: "circle"` (§7) at the ellipse's own square-ish bounding
box — don't split it into a separate ellipse-mask shape, `mask_name` does the clipping natively.

**When the root FRAME has no explicit `dimensions` in `get_figma_data`** (seen when the frame's own
sizing is `{"mode": "none", "sizing": {}}` with nothing else) — don't guess the canvas size from the
children's bounding box (background/decorative elements often bleed off-canvas, which throws off a
bbox guess). Instead, `download_figma_images` the root frame node itself at `pngScale: 1` — Figma
always rasterizes a frame at its own true defined size, so the returned pixel dimensions are exact.
This same rendered PNG also doubles as a visual reference to check your element mapping against.

**Finding fonts efficiently**: `list_fonts` has ~1150 Google fonts plus the workspace's custom
uploads — always pass the `name` filter (matching ignores case, accents, spaces and hyphens) instead
of paging through the full list, e.g. `list_fonts(name: "Open Sans")`, or `category` (e.g. `serif`)
to find a substitute for an unavailable Figma font. Filter a weight with `weight` (700 = bold) and
italic with `style: "italic"` — not by putting "Bold"/"Italic" in the name.

## 7. Abyssale `import_design_from_json` schema gotchas (learned the hard way)

- The payload needs `name`, `type` and `target` — `target.project_uuid` must be a real project of the
  user's (`list_projects`, or `create_project` for a new one). Never leave the `YOUR_PROJECT_UUID`
  placeholder from `get_design_import_example`.
- A multi-format design must key **every** layer's `layout` and `properties` by format name (a
  `default` key can hold shared values); a flat shape is only valid with a single format.
- `layout.<format>.x/y/width/height` must be **integers** — round everything, `wrong_type` error otherwise.
- Colors: hex strings (`#RGB`/`#RGBA`/`#RRGGBB`/`#RRGGBBAA`), `cmyk(…)` strings, or gradient **strings**
  (see the gradient bullet below) — never a 10-char hex string, never an object.
- `text` `font_size` is required (2–1000) and is a starting size — the text shrinks to fit its box;
  `font_weight` is a multiple of 100 (100–900).
- `stroke` on `shape`/`button` is **nested**: `{"color": ..., "width": ...}`, not flat `stroke_color`/`stroke_width`.
- `padding` on `button` is nested `{"vertical": ..., "horizontal": ...}` — and it is not
  optional in practice: **`padding` drives the button's actual rendered size, not
  `layout.width/height`.** A button with no `padding` set renders far smaller than its
  declared `width`/`height` — confirmed side by side: the same `layout` box (100×44),
  one button with no `padding` rendered barely bigger than its own text, the other with
  `padding: {"vertical": 14, "horizontal": 20}` rendered at the intended size. Always
  set an explicit `padding` on every `button` layer; treat the declared `width`/`height`
  as a rough positioning hint, not a guaranteed final size — a button that sits close to
  a canvas edge or another element may need re-checking after a real render, since its
  actual rendered width isn't precisely predictable in advance.
- `rotation` (shape/button) must be **0–360**, not signed degrees.
- `scale` in animation tweens must stay **0–100**.
- A layer allows only **one tween per type** — repeated effects need multiple keyframe pairs packed into one tween's `keyframes` array.
- `video` layer type rejects `fitting_type`/`alignment` (unlike `image`).
- `fitting_type` on `image` only accepts `"cover"` or `"fill"` — never `"contain"`.
- `mask_name: "circle"` is available on `image` layers for circular crops (other masks: `rounded_corners` — needs `mask_properties.radius` —, `blob`, `squircle`, `pentagon`, `hexagon`, `parallelogram`, `window`).
- **`radius` works on `image` layers too** (not just `shape`) — confirmed via `validate_only`, e.g. `radius: 284` on a 568×745 image produces a pill/stadium shape (rounded rect where radius ≈ half the shorter side), not just a plain circle mask. Use this for any Figma `RECTANGLE` with an image fill and a large `borderRadius`.
- **`rotation` and `shadow` both work on `text` layers**, not just `shape`/`button` (undocumented previously, confirmed via `validate_only` + a real render). `rotation` is clockwise, 0–360, and pivots near the **top-left corner of the declared layout box** — confirmed empirically: a box `{x:100,y:100,w:200,h:40}` with `rotation: 90` rendered as a vertical strip reading bottom-to-top, anchored near the box's original top-left, not its center. For a Figma node whose reported bounding box is already the *rotated* box (e.g. a vertical text label with `w:42,h:530`), swap back to the *pre-rotation* horizontal dimensions (`w:530,h:42`) and keep the same `x,y` origin before applying `rotation: 90` — don't feed it the already-rotated box.
- **Gradients are native, as strings — not objects.** Passing an object for `background_color` (e.g. `{"type": "radial", "colors": [...]}`) fails with `wrong_type: Not a valid string`. Use the string form instead: `linear-gradient(<x>% <y>% <x1>% <y1>%,<off>% <color> <alpha>,…)` or `radial-gradient(<cx>% <cy>% <r>%,<off>% <color> <alpha>,…)`, with 2 to 8 stops (offset 0–100%, alpha 0–1). It's accepted on a text `color`, a shape/button `background_color` and the format background on static and animated designs (printer designs: shape/button background only). For a Figma `GRADIENT_LINEAR`/`GRADIENT_RADIAL` fill, convert `fills[].gradient` into that string. Only fall back to a rasterized PNG (`download_figma_images` of the node) for effects the string can't express.

## 6c. Elements positioned off-canvas — exclude entirely, don't decompose

Figma files sometimes carry leftover/duplicate assets positioned thousands of pixels outside the
frame (seen: `x: -8916` on an 1080-wide canvas) — remnants of a component library or an old layout
pass. These never render and are not part of the visible design. Check every node's absolute
position against the canvas bounds; anything with **zero overlap with the canvas rectangle** should
be dropped from the payload entirely. This is not a "no grouping" violation — an invisible node
isn't a visual element to preserve.

## 6d. Verify visually before assuming a transform (rotation, tiling, etc.)

Don't infer rotation/skew from coordinates alone — download the target node (or its parent frame)
and `Read` it first. A real case: a repeating background pattern of 16 text tiles with staggered
x/y offsets looked like it *could* be a diagonal rotated tiling from the coordinates alone, but the
rendered reference showed it was actually a plain axis-aligned staggered/brick grid (no rotation at
all, `rotation` field wasn't even present on those nodes) — one reference-image check avoided
wasting a `rotation` guess on 16 layers.

## 6e. Device/UI mockups (phone frames, browser chrome, etc.)

Same principle as complex `INSTANCE` chrome (§4): decompose into the distinct real layers rather
than one flattened image — typically **bezel/frame vector** (image), **artwork/screen content**
(image, `imageRef` from the inner content rectangle), and if present a **top overlay frame** (image,
drawn last/on top — handles rounded-corner clipping/notch cutout that the bare artwork would
otherwise bleed past). Skip a plain white/solid "mask" base layer that sits directly behind
already-opaque cover-fit artwork of the exact same box — it can never be visible, so it isn't a
decomposition shortcut to include it, just dead weight.

## 6f. Verifying a created design: the Abyssale app editor URL isn't fetchable

A design's page in the Abyssale web app is a JS-rendered SPA behind auth — `WebFetch` on
it only ever returns a static loading-shell message ("Abyssale is loading your experience..."), never
the actual design content. Don't rely on it to verify a design. Always use
`generate_static_banner` + download + `Read` (§8 step 9) for visual QA instead.

## 8. Full pipeline, step by step (this is exactly what worked, repeatedly)

1. Extract `fileKey` + `nodeId` from the Figma URL (`node-id=942-2723` → nodeId `"942:2723"`, dashes→colons).
2. `get_figma_data(fileKey, nodeId)`. If node has no `node-id` in URL, first call with `depth: 2` to find the right top-level frame.
3. Map every node per §4, compute absolute positions per §3, resolve colors/fonts per §5.
4. For nodes needing real pixels: `download_figma_images(fileKey, nodes: [{nodeId, imageRef?, fileName}], localPath: "src/assets/figma", pngScale)`.
   - For hug-sized nodes with no explicit Figma dimensions, the downloaded pixel size ÷ pngScale gives you the missing layout box.
5. Build the `import_design_from_json` payload (`name`, `type`, `target.project_uuid` from `list_projects`/`create_project`, `formats`, `layers`). Start from `get_design_import_example` for the exact shape.
6. `import_design_from_json(validate_only: true)` — fix any schema errors (see §7), repeat until clean. A clean result creates nothing and doesn't check fonts, image URLs or the project.
7. `import_design_from_json(validate_only: false)`. Images: an image `src` given as a public HTTPS URL (or a `data:` URI of at most 100 KB) needs no upload — the call waits up to ~60 s and usually returns `DONE` or `FAILED` directly. The downloaded Figma assets are local files, so with a bare filename `src` it returns `WAITING_FOR_VALIDATION` with one upload command (curl) **per image layer**, only in this first response (status checks never repeat them). Each has a **unique** `policy` + `signature` — never reuse one file's signature for another file. Run each one yourself, verbatim except for `<LOCAL_FILE_PATH>` (don't reorder its fields), within one hour; never ask the user to run them. Every format of a layer must use the same file. (Claude.ai additionally needs the Abyssale upload bucket on its domain allowlist — see the MCP overview docs.)
8. If the import isn't finalized yet, call `check_design_import_status(import_id)` (after uploads, this call is what starts processing) until `DONE` → `result.uuid` is the new Abyssale design id; check `warnings` (e.g. `text_fit_failed`, `font_weight_adjusted`). `QUEUED`/`PROCESSING` are normal; `FAILED` is final — read `error`. Usually 1-2 checks with a few seconds' wait are enough; don't poll indefinitely.
9. `generate_static_banner(design_id, template_format_name)` (this spends credits) → download the rendered image and `Read` it to visually QA against the Figma source before reporting back.
10. Report to the user: what got natively decomposed vs. what got flattened to an image (and why), plus any visual defects spotted in the render.

## 9. Standing rules from the user (do not relitigate)

- Default to full per-element decomposition — no silent flattening/grouping of distinct visual
  elements into one image. Any flatten-to-image decision on a complex instance must be disclosed,
  not decided silently.
- Ask before taking initiative beyond what was asked (no unsolicited reports/artifacts/features).
- Give concise, step-by-step, verifiable answers — verify claims (read source/docs) rather than
  speculate, especially about tool/API behavior.
