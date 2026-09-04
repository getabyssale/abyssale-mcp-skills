---
name: html-to-abyssale
description: Recreate an HTML/CSS banner (absolute-positioned elements over a fixed-size canvas — the shape of HTML5 ad banners, email banners, or design-tool HTML exports) as a native, editable Abyssale design, including basic CSS/GSAP animation mapped to Abyssale's tween system. Use when the user gives HTML/CSS for a banner and wants it rebuilt in Abyssale via import_design_from_json, not just referenced. Does not handle canvas-rendered banners; animation support is best-effort, not a full reproduction of every CSS/GSAP effect.
---

# HTML → Abyssale conversion

Goal: read an HTML/CSS banner's structure and rebuild it as native Abyssale layers
(text/button/shape/image), one element per visible node — not flattened into a
screenshot. Animation is carried over on a best-effort basis: reach for the closest
native Abyssale tween, don't try to reproduce every CSS/GSAP effect exactly.

## Scope

**Handles**: absolute-positioned banners over a fixed-size canvas — the structure
HTML5 ad banners, email banners, and design-tool exports actually use (a container
with a fixed `width`/`height`, children `position: absolute` with `left`/`top`).

**Does not handle: canvas-rendered banners.** Some real HTML5 ad banners render
everything by drawing onto one or more `<canvas>` elements from JavaScript, with no
DOM structure to inspect — no `left`/`top`, no text nodes, just imperative draw calls
inside `.js` files. Detect this by checking whether the visible content lives inside
`<canvas>` tags with empty bodies (as opposed to `<img>`/`<div>` children). If so, say
so and stop — don't attempt a conversion, and don't fall back to a screenshot import
without the user explicitly asking for that specific (much lower-fidelity) outcome.

**No single fixed canvas (flexbox-centered `body`, no explicit `width`/`height` on the
container)**: don't hand-estimate pixel positions from reading the CSS — guessed
coordinates have been wrong enough to cause visible overlaps. Measure real positions
instead (see "Measuring real positions" below).

**A source with a `<video>` element** maps to a native `video` layer — just a plain
hosted `src` (no other required properties), positioned/sized like any other layer,
confirmed via a real conversion (a hero `<video autoplay muted loop>` → one `video`
layer with `properties.default.src` set to the same hosted `.mp4` URL). Like every
other layer, it still needs the full-duration `animation` block from the rule below
(the official `animated-video` example gives it one even with no `tweens`) — a video
layer isn't exempt from that. Get the exact shape fresh with
`get_design_import_example(type: "animated-video")` rather than reconstructing it.

**One HTML source becoming a multi-format Abyssale design** — several formats
sharing one layer set, each format overriding `layout` (and, where needed,
`properties`) — is confirmed working: build `formats` as a list of `{name, width,
height, background_color}` entries, and give every layer a `layout.<format>` (and, if
the format's box materially changes proportions, redesign that layer's numbers for the
new box rather than naively scaling — a 320×480 hero video area doesn't have to stay
the same height in a 300×600 half-page, and the official `static-multi-format`/
`animated-video` examples both hand-tune per-format numbers rather than scale). See
"Multi-format and `slide` tweens don't mix safely" below before animating layers whose
`layout` differs by format.

## Steps

0. **Before writing any payload by hand, fetch a fresh official example** —
   `get_design_import_example(type: "animated")` (or `"static"`, `"animated-video"`,
   `"static-multi-format"` as fits) — rather than reconstructing the shape from memory
   or an earlier conversation summary. Several real mistakes this skill's own rules
   below were learned from (`formats` built as an object instead of a list, a
   top-level `animations` map instead of a per-layer `animation` block, tween
   keyframes missing their `data.time`/`data.type`/`ease` structure) came specifically
   from reconstructing the schema from memory instead of re-reading a real example
   first. The example is the source of truth; this skill's notes are a supplement to
   it, not a replacement.

1. **Find the canvas.** Locate the outer container with an explicit fixed
   `width`/`height` (in px) and its `background-color` — this becomes one entry in
   the Abyssale payload's `formats` **list** (`"formats": [{"name": ..., "width": ...,
   "height": ..., "background_color": ...}]`) — **a list, not an object keyed by
   format name**: sending `{"formats": {"custom-300x250": {...}}}` fails validation
   with `formats: Not a valid list`, a mistake easy to make since `layout` and
   `properties` on each *layer* genuinely are keyed-by-format-name objects — `formats`
   at the payload root is the one exception, always an array.

2. **Map each visible child to a layer**, using its own `left`/`top`/`width`/`height`
   directly as `layout.<format>.{x,y,width,height}` — no coordinate accumulation
   needed, since CSS absolute positioning within the fixed canvas already gives final
   coordinates. Skip purely structural wrappers with no visible fill/text/image.

   | CSS element | Abyssale layer |
   |---|---|
   | A `<div>`/text node with just text content | `text` — `payload`, `color`, `font`, `font_size`, `font_weight`, `text_align`, `vertical_align`. A source `<br>` or hard line break becomes a literal `\n` inside `payload` (confirmed: `"payload": "PUSH\nHARDER"` rendered as two lines, matching the source's `PUSH<br>HARDER`) — don't try to split it into two separate text layers. |
   | A `<div>` styled as a button/pill (background + radius + text, e.g. a CTA) | `button` — use this native type directly, don't decompose into shape+text. Same text properties, plus `background_color`, `radius`, and a required nested `padding: {vertical, horizontal}` — see the note right after this table. **No `vertical_align`** — including it fails validation with `unknown_field`. |
   | A `<div>` styled as a filled/bordered box with no text | `shape` — `background_color`, `radius`, optional nested `stroke: {color, width}` |
   | An `<img>`, or a CSS `background-image` on a container | `image` — see `abyssale-import` for supplying the source (hosted URL preferred, inline base64 for a small asset, or the upload-command flow for a local file) |
   | An inline `<svg>` | No native SVG/vector layer type exists — rasterize it (e.g. with `cairosvg`) and import as one `image` layer. Don't try to decompose an SVG's sub-elements into separate Abyssale layers unless there's a real reason to animate them independently — flatten it as one piece by default |
   | Text baked into a full-canvas transparent PNG rather than a live DOM text node (seen on real production banners using custom stroked/bevelled typography no web font could reproduce) | `image`, not `text` — there's no live font/color/content to extract |
   | A `<video>` element | `video` — `properties.default.src` set to a hosted `.mp4` URL, `layout` as usual. Confirmed working; see the Scope section above. |

**Text auto-wraps by declared `width` — a manual `\n` is only needed for a source
`<br>`.** Confirmed on a headline with no line break in its source markup at all: given
a `width` narrower than the text's natural single-line size, Abyssale wrapped it to two
lines on its own, and did so identically across two formats whose `layout` declared
different widths for the same layer — no `\n` needed, and none should be added, when
the source doesn't have an explicit break.

**`button` sizing is driven by `padding`, not by `layout.width/height`.** A `button`
layer with no `padding` set renders far smaller than its declared box — confirmed
side by side on the same declared 100×44 box: with no `padding`, the button rendered
barely larger than its own text; with `padding: {"vertical": 14, "horizontal": 20}`,
it rendered at the intended size. Always set an explicit `padding` derived from the
source CSS (its own `padding`, or a reasonable estimate from the box size and font
size if the source used flex-centering instead). Treat the declared `width`/`height`
as an approximate positioning hint, not a guaranteed final render size: a button
placed close to a canvas edge or another element (e.g. an icon positioned just past
where you expect the button to end) may need a re-check after the real render, since
its actual rendered width isn't precisely predictable from `padding` and text content
alone before rendering.

3. **Colors**: CSS hex (`#RRGGBB`) maps directly. Convert `rgb()`/`rgba()` to hex
   (`#RRGGBBAA` with alpha) — Abyssale never takes a CSS color function string.

4. **Fonts**: Abyssale needs a font **uuid**, not a family name — call `list_fonts`
   filtered by `name`. **Prefer a Google Fonts result (`type: "google"`) over a
   custom-uploaded one of the same name** if both exist: a custom upload can have
   different internal line-height/ascent metrics, which has caused real, confirmed
   text-box overlap at coordinates that render cleanly with the Google Fonts version.
   Font weight isn't validated against the font's `available_weights` — an
   unavailable weight is silently substituted to the nearest one. That substitution
   is reported as a `warnings` entry (`font_weight_adjusted`) on a **real** import
   call, but not on `validate_only` — check `warnings` after the real import, don't
   rely on a clean `validate_only` to mean no substitution happened.

5. **Validate, then import for real**: `import_design_from_json(validate_only: true)`
   first, fix schema errors, then `validate_only: false`. Only the real import's
   `warnings` array surfaces a font-weight substitution. A real import can also come
   back with a `preview_generation_failed` warning (`"1 of 1 preview(s) failed to
   render; the import completed without them"`) — confirmed benign: the design is
   still created and finalized (`is_finalized: true`), and rendering it explicitly via
   `generate_banners_async`/`generate_static_banner` right after works normally. Don't
   treat this warning as a reason to re-import; treat it as a reason to do step 6.

6. **Visually QA** — `generate_static_banner` for a static design; for `animated`
   designs it rejects outright (`template_not_static`), use `generate_banners_async`
   with `image_file_type: "mp4"` instead, download, and pull a few frames at
   different timestamps (`ffmpeg -ss <t> -i render.mp4 -frames:v 1 out.jpg`) to
   confirm animation actually progresses — don't trust a clean `validate_only` alone.
   Pull a frame near the **end** of the timeline specifically, not just mid-animation
   ones: a layer whose `start_at_s`/`end_at_s` window was mistakenly scoped to its own
   tween (see above) renders fine early on and only reveals the bug by vanishing late,
   which a check at t=0 and t=mid alone won't catch.
   A plain solid-color shape's rotation can't be visually confirmed this way (no
   asymmetry to judge by) — use a shape with a visible feature, or note plainly that
   rotation wasn't verified.

## Animation — best effort, not a full reproduction

If the source animates — CSS `@keyframes`/`transition`, or a GSAP timeline — set the
design's `type` to `"animated"` and add **two** animation blocks: one at the payload
**root** (`"animation": {"duration": <seconds>, "screenshot_at_s": <seconds>}`) and
one **per animated layer** (`"animation": {"start_at_s", "end_at_s", "tweens": [...]}`).
Omitting the root one fails with `missing_required`.

**A layer's `start_at_s`/`end_at_s` is its presence window on the timeline, not just
its tween's active span — get this wrong and the element vanishes right when its
fade/slide-in finishes, instead of staying visible.** Confirmed by rendering a design
where every layer's `start_at_s`/`end_at_s` was set to match its own fade-in's timing
(e.g. `0.1`–`0.7` for a layer whose tween ran 0.1→0.7): the layer was only ever
rendered *inside* that window — it wasn't there before `start_at_s`, and it
disappeared again the instant `end_at_s` passed, discarding the fade-in's whole point
(that the element should then stay on screen). Pulling frames from the rendered mp4 at
increasing timestamps showed elements popping out one by one as their individual
windows closed, until only the last-arriving layer remained. **Always set
`start_at_s: 0` and `end_at_s: <root duration>` on every layer** (matching the
official multi-format example, where every layer spans the full timeline even though
its tween only moves during a sub-range of it) — the tween's own keyframe `data.time`
values are what stagger the actual movement inside that constant presence window; the
engine holds the first keyframe's value before the tween starts and the last
keyframe's value after it ends.

**Widening a button layer to a full-duration window can then trigger a `cannot fit`
render error that an identical narrower-window version didn't have** — confirmed by
isolating the change to one `button` layer, same `payload`/`padding`/`font_size`,
only the window widened: the narrow-window version rendered fine, the wide-window one
failed with `Element <name> error: The text '<payload>' cannot fit within the defined
space.`, reproducibly, across repeated renders. Cause (inferred from the fix that
resolved it, not confirmed from Abyssale internals): a full-duration window puts the
button in a static "resting" pose before its tween starts, and in that pose the
declared `layout.width` seems to be enforced as a hard cap that `padding`-driven
auto-sizing (see the padding note above) doesn't get to override — unlike during the
tween's own active span, where it rendered fine at the same padding. **Fix: give an
animated button's declared `layout.width` real margin beyond `padding.horizontal * 2 +
estimated text width`, don't rely on padding alone once the layer spans the full
timeline** — widening one button from 130px to 175px (same text/padding/font)
resolved it with no other change.

**Tween vocabulary** (confirmed against the official reference,
developers.abyssale.com/rest-api/designs/import/animated):

| Tween type | Keyframe `attr` | Range |
|---|---|---|
| `slide` | `left`, `top` (px, both required) | signed px |
| `scale` | `scale` (+ optional `scale_origin_x`/`scale_origin_y`) | **0–100, hard-capped — 100 is the ceiling, not just "as designed."** A keyframe above 100 fails validation with `out_of_range` (confirmed: `115` rejected outright). A CSS pulse that overshoots past 100% (e.g. `scale(1)` → `scale(1.15)`) has no direct equivalent — remap it as a grow-in *up to* 100 instead (e.g. `88` → `100`), never as `100` → `>100`. |
| `rotate` | `angle` | unbounded signed degrees, not clamped to ±360 |
| `fade` | `opacity` | 0.0–1.0 (a fraction — unlike `scale`'s 0–100) |
| `audioFade` | `volumeEffect` | 0.0–1.0 |
| `textEffect` | `typewriting` (0 or 100 only) + `textEffectType` (`classic`, `softFade`, `focusIn`, `riseUp`, `riseDown`) | — |

GSAP ease strings carry over directly into `ease`, **including parametrized ones**
(`"back.out(1.7)"`, `"elastic.out(1, 0.5)"` both validate and render correctly).

**Multi-format and `slide` tweens don't mix safely.** A layer's `tweens` are declared
*once*, not per format, but a `slide` tween's `attr.left`/`attr.top` are absolute
canvas pixels — and if that layer's `layout` differs by format (the normal case for a
multi-format design), those absolute values can only be correct for one format's box.
Whether the engine silently reinterprets them per-format or just applies the same
pixels everywhere is **not confirmed either way** — it was sidestepped, not tested, by
avoiding `slide` entirely on any layer with per-format `layout` in the one multi-format
animated design built so far (rotate/scale/fade were used instead throughout, since
none of those three reference a position and so can't go wrong across formats). Do the
same unless someone actually confirms the real behavior: on a multi-format animated
design, restrict `slide` to a layer whose `layout` box is byte-identical across every
format, and use rotate/scale/fade for anything whose position varies by format.

**`.from()` is not "`.to()` reversed" — check what it actually animates FROM and TO.**
GSAP's `.from(el, {opacity: 1, ...})` animates from the given values TO the element's
own resting CSS state. If that resting state is `opacity: 0` (e.g. a `.hidden` class),
the call is a flash-then-fade-**out**, not a fade-in — the opposite of what those same
values mean on `.to()`. Don't "normalize" a `.from()` call to match its `.to()`
siblings without checking the element's CSS resting state first — assuming it must be
an authoring mistake produced a real, confusing bug (an element stayed at opacity 1
for the rest of the timeline instead of correctly disappearing, ghosting behind every
later scene).

**A "hold"** (appears, stays, then disappears) is a validated pattern: pack 4
keyframes into one `fade` tween — `[opacity:0 @ start]`, `[opacity:1 @ fade-in end]`,
`[opacity:1 @ hold end — same value, later time]`, `[opacity:0 @ fade-out end]`. If
several layers share one box and take turns being visible (a sequential multi-scene
banner), give each its own hold-pattern tween with non-overlapping visible windows —
compute absolute times from the GSAP timeline by hand (`"+=N"` adds `N`s after the
current end of the timeline; `"<"` starts alongside the immediately-preceding tween's
start). If two scenes end up visible at once, the likely cause is a layer that never
actually reaches opacity 0 — re-check its keyframes (and its `.from()`/`.to()` intent)
before assuming it's an Abyssale bug.

### Effects with no native equivalent — approximate simply, don't over-engineer

None of these have a native Abyssale tween. Use the closest single native
approximation and move on — don't build an elaborate multi-layer or multi-keyframe
reproduction for the sake of fidelity:

- **A curved/bezier motion path** → a plain `slide` from the path's start point to its
  end point (skip the intermediate waypoints unless there's a strong reason to keep
  them — a straight line is already a reasonable best-effort stand-in).
- **`repeat`/`yoyo` (looping)** → play it once. Only chain a couple of extra keyframe
  pairs manually (e.g. `angle` 0→360→720) if a short, obviously-bounded repeat is
  clearly the point of the effect — never attempt to fake a long or indefinite loop.
- **`filter: blur()`, `backdrop-filter`, glassmorphism, complex gradients** → don't
  approximate with a flat-color Abyssale shape, it reads as visibly wrong rather than
  simplified. If the effect is purely decorative, it's fine to drop it. If it matters
  to the design, rasterize just that portion with a headless browser
  (`google-chrome --headless=new --screenshot=out.png "file://<path>"`, with the
  layers you're keeping native hidden via injected `visibility: hidden` CSS first)
  and import the result as one `image` layer.
- **Per-character text stagger** → animate the whole text block as one unit; the
  stagger itself is lost, not approximated.
- **3D transforms (`rotateX`/`rotateY`)** → drop the 3D aspect, keep any accompanying
  2D motion (fade/slide) on the same element.
- **Multiple independently-animated pieces inside one SVG** → flatten the whole SVG
  to one rasterized `image` layer and apply a single simple tween to it (fade/scale
  in, say) rather than rasterizing each sub-element separately to preserve individual
  motion — that level of decomposition is not worth it for a best-effort pass.

## Measuring real positions (non-fixed-canvas layouts)

1. Inject a script before `</body>` that, after a `setTimeout` matching the
   animation's duration, calls `getBoundingClientRect()` on the relevant elements and
   writes the result as JSON into a `<pre id="__measurements">` appended to `body`.
2. `google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=<ms>
   --window-size=<w>,<h> --dump-dom "file://<path>"` — the time budget fast-forwards
   past the injected `setTimeout`.
3. Grep the measurements out of the dumped HTML and use those as `layout` coordinates.

The `--window-size` you choose is the layout's real basis (it decides whether a CSS
`max-width` kicks in) — pick one deliberately.

**`--screenshot` can silently truncate the paint well short of the requested
`--window-size`, independent of your page's content.** Confirmed on this environment
with a single flat, fully-static `<div>` sized exactly to `--window-size`: the bottom
portion of the image came back plain white instead of the div's background color, even
though an injected script confirmed the div's `getBoundingClientRect()` height matched
the window exactly. Requesting a **taller** window than the actual content needs (e.g.
`--window-size=<w>,<h+200>`) and cropping the result down to `<w>x<h>` afterward made
the same content paint correctly end to end — so when a `--screenshot` capture looks
suspiciously cut off, don't assume it's a real CSS/layout bug in the page: verify with
a computed-style probe (inject a script that reports `getBoundingClientRect()`/
`getComputedStyle()` into a `<pre>`, then `--dump-dom` — see steps 1–2 above) before
"fixing" a layout that may already be correct.

## Notes

- CSS `line-height` in `em`/unitless form converts to Abyssale's percentage form
  (`1.4` → `140`).
- An unknown property name on a layer is a hard validation error (`unknown_field`) —
  don't assume every part of the API silently ignores unrecognized keys.
- Uploading a local asset for a real import (rather than inlining base64) is the more
  reliable default once a file is more than trivially small — a long base64 string is
  easy to corrupt when it has to be reproduced across tool calls, where a bare
  filename plus the upload-command flow (see `abyssale-import`) isn't.
