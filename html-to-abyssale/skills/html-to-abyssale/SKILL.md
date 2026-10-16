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
hosted `src` (its only required property; `muted` and `opacity` are the only other
ones — map a source `muted` attribute to `muted: true`), positioned/sized like any
other layer, confirmed via a real conversion (a hero `<video autoplay muted loop>` →
one `video` layer with `properties.default.src` set to the same hosted `.mp4` URL).
Like any layer, it spans the whole timeline unless you give it an `animation` window
(see the animation rules below). Get the exact shape fresh with
`get_design_import_example(type: "animated-video")` rather than reconstructing it.

**One HTML source becoming a multi-format Abyssale design** — several formats
sharing one layer set — is confirmed working: build `formats` as a list of `{name,
width, height, background_color}` entries, and key **every** layer's `layout` **and**
`properties` by format name (a `default` key can hold the values shared by all
formats; a flat `properties` is rejected once there are two formats). If
the format's box materially changes proportions, redesign that layer's numbers for the
new box rather than naively scaling — a 320×480 hero video area doesn't have to stay
the same height in a 300×600 half-page, and the official `static-multi-format`/
`animated-video` examples both hand-tune per-format numbers rather than scale. See
"Multi-format and `slide` tweens don't mix safely" below before animating layers whose
`layout` differs by format.

## Steps

0. **Before writing any payload by hand, fetch a fresh official example** —
   `get_design_import_example(type: "animated")` (or `"static"`, `"animated-video"`,
   `"static-multi-format"` as fits) — rather than reconstructing the shape from memory
   or an earlier conversation summary — common mistakes otherwise: `formats` built as
   an object instead of a list, a top-level `animations` map instead of a per-layer
   `animation` block, tween keyframes missing their `data.time`. The example is the
   source of truth; this skill's notes are a supplement to it, not a replacement. Every example contains `target.project_uuid:
   "YOUR_PROJECT_UUID"` — replace it with a real project: call `list_projects` and ask
   the user which one, or `create_project` for a new one.

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
   | A `<div>`/text node with just text content | `text` — `payload`, `color`, `font`, `font_size` (required, 2–1000 — a starting size that auto-shrinks to fit the box), `font_weight`, `text_align`, `vertical_align`. A source `<br>` or hard line break becomes a literal `\n` inside `payload` (e.g. `PUSH<br>HARDER` → `"payload": "PUSH\nHARDER"`) — don't try to split it into two separate text layers. |
   | A `<div>` styled as a button/pill (background + radius + text, e.g. a CTA) | `button` — use this native type directly, don't decompose into shape+text. Same text properties, plus `background_color`, `radius`, `padding` (a number or `{vertical, horizontal}` — always set it, see the note right after this table) and `alignment` (where the button sits inside its layer box, default `middle center`). **No `vertical_align`** — including it fails validation with `unknown_field`. |
   | A `<div>` styled as a filled/bordered box with no text | `shape` — `background_color`, `radius`, optional nested `stroke: {color, width}` |
   | An `<img>`, or a CSS `background-image` on a container | `image` — `src` as a public HTTPS URL (preferred); otherwise a `data:image/...;base64,` URI of at most 100 KB (tell the user if you recompressed it); a bare local filename only if you can run shell commands — see `abyssale-import` for that upload flow |
   | An inline `<svg>` | No native SVG/vector layer type exists — rasterize it (e.g. with `cairosvg`) and import as one `image` layer. Don't try to decompose an SVG's sub-elements into separate Abyssale layers unless there's a real reason to animate them independently — flatten it as one piece by default |
   | Text baked into a full-canvas transparent PNG rather than a live DOM text node (seen on real production banners using custom stroked/bevelled typography no web font could reproduce) | `image`, not `text` — there's no live font/color/content to extract |
   | A `<video>` element | `video` — `properties.default.src` set to a hosted `.mp4` URL, `layout` as usual. Confirmed working; see the Scope section above. |

**Text auto-wraps by declared `width` — a manual `\n` is only needed for a source
`<br>`.** Confirmed on a headline with no line break in its source markup at all: given
a `width` narrower than the text's natural single-line size, Abyssale wrapped it to two
lines on its own, and did so identically across two formats whose `layout` declared
different widths for the same layer — no `\n` needed, and none should be added, when
the source doesn't have an explicit break.

**A `button`'s visible box is sized from its label plus `padding`, and placed inside
the layer box by `alignment`.** Always set an explicit `padding` derived from the
source CSS (its own `padding`, or an estimate from the box size and font size if the
source used flex-centering). With the default `alignment` (`middle center`) the button
is centered in its layer box — use e.g. `"middle left"` to match a CSS box anchored on
its left edge. The layer box must be large enough for the label: if the text can't fit,
the render fails — give the box enough width, or set `min_font_size` if the label may
shrink. Check buttons near a canvas edge or another element on the first render.

3. **Colors**: CSS hex maps directly (`#RGB`, `#RGBA`, `#RRGGBB`, `#RRGGBBAA`).
   Convert `rgb()`/`rgba()`/`hsl()` to hex (`#RRGGBBAA` with alpha) — those functions
   are rejected. Linear and radial **gradients are native** as strings —
   `linear-gradient(<x>% <y>% <x1>% <y1>%,<off>% <color> <alpha>,…)` /
   `radial-gradient(<cx>% <cy>% <r>%,<off>% <color> <alpha>,…)`, 2–8 stops — on a
   text `color`, a shape/button `background_color` and the format background. Check the
   exact accepted syntax in the validation error if one is rejected.

4. **Fonts**: Abyssale needs a font **uuid**, not a family name — call `list_fonts`
   filtered by `name`. If both a Google Fonts result (`type: "google"`) and a custom
   upload of the same name exist, prefer the Google Fonts one unless the brand requires
   its own file — then check the first render for spacing. A weight the font doesn't
   have is replaced by the nearest one, reported as a `font_weight_adjusted` warning on
   the real import (`validate_only` doesn't resolve fonts).

5. **Validate, then import for real**: `import_design_from_json(validate_only: true)`
   first, fix schema errors, then `validate_only: false`. A clean `validate_only`
   creates nothing and doesn't check fonts, image URLs or the project — only the real
   import's `warnings` array surfaces a font-weight substitution or a text that didn't
   fit (`text_fit_failed`; pass `strict: true` to make that a failure instead). The real
   import waits up to ~60 s and usually returns the final status; if it comes back with
   `is_finalized: false`, check again with `check_design_import_status` (statuses:
   `WAITING_FOR_VALIDATION`, `QUEUED`, `PROCESSING`, `DONE`, `FAILED`). On `DONE` the
   design id is `result.uuid`; on `FAILED` read `error`. Read every warning, then
   render the design in step 6.

6. **Visually QA** (each render spends Abyssale credits) — `generate_static_banner`
   for a static design; for `animated` designs it rejects outright
   (`template_not_static`), use `generate_banners_async` with `image_file_type: "mp4"`
   instead (it waits up to ~60 s and usually returns the file URL directly). If you can
   run shell commands, download it and pull a few frames at different timestamps
   (`ffmpeg -ss <t> -i render.mp4 -frames:v 1 out.jpg`) to confirm animation actually
   progresses — don't trust a clean `validate_only` alone. Without a shell or ffmpeg,
   give the user the mp4 URL and say plainly that frame-by-frame QA wasn't done.
   Pull a frame near the **end** of the timeline specifically, not just mid-animation
   ones: a layer whose `start_at_s`/`end_at_s` window was mistakenly scoped to its own
   tween (see below) renders fine early on and only reveals the bug by vanishing late,
   which a check at t=0 and t=mid alone won't catch.
   A plain solid-color shape's rotation can't be visually confirmed this way (no
   asymmetry to judge by) — use a shape with a visible feature, or note plainly that
   rotation wasn't verified.

## Animation — best effort, not a full reproduction

If the source animates — CSS `@keyframes`/`transition`, or a GSAP timeline — set the
design's `type` to `"animated"` and add an animation block at the payload **root**
(`"animation": {"duration": <seconds>, "screenshot_at_s": <seconds>}` — `duration` is
required, 1–60 s; `screenshot_at_s` is optional and defaults to the duration) plus one
**per animated layer** (`"animation": {"start_at_s", "end_at_s", "tweens": [...]}`).
Omitting the root one fails with `missing_required`. A layer with no `animation`
block simply spans the whole timeline. Animated formats must fit within 1920×1200,
and animated designs require a plan that includes them (`feature_not_in_plan`
otherwise).

**A layer's `start_at_s`/`end_at_s` is its presence window on the timeline, not just
its tween's active span — get this wrong and the element vanishes right when its
fade/slide-in finishes, instead of staying visible.** Confirmed by rendering a design
where every layer's `start_at_s`/`end_at_s` was set to match its own fade-in's timing
(e.g. `0.1`–`0.7` for a layer whose tween ran 0.1→0.7): the layer was only ever
rendered *inside* that window — it wasn't there before `start_at_s`, and it
disappeared again the instant `end_at_s` passed, discarding the fade-in's whole point
(that the element should then stay on screen). Pulling frames from the rendered mp4 at
increasing timestamps showed elements popping out one by one as their individual
windows closed, until only the last-arriving layer remained. **Leave the window out
(it defaults to `start_at_s: 0` / `end_at_s: <root duration>`), or set exactly those
values, on every layer that should stay visible** (matching the
official multi-format example, where every layer spans the full timeline even though
its tween only moves during a sub-range of it) — the tween's own keyframe `data.time`
values are what stagger the actual movement inside that constant presence window; the
engine holds the first keyframe's value before the tween starts and the last
keyframe's value after it ends.

**Tween vocabulary** (confirmed against the official reference,
developers.abyssale.com/rest-api/designs/import/animated):

| Tween type | Keyframe `attr` | Range |
|---|---|---|
| `slide` | `left`, `top` (px, both required) | signed px |
| `scale` | `scale` (the tween itself can also take `scale_origin_x` — `left`/`center`/`right` — and `scale_origin_y` — `top`/`middle`/`bottom` — always as a pair; they're tween fields, not keyframe attrs) | **0–100, hard-capped — 100 is the ceiling, not just "as designed."** A keyframe above 100 fails validation with `out_of_range` (confirmed: `115` rejected outright). A CSS pulse that overshoots past 100% (e.g. `scale(1)` → `scale(1.15)`) has no direct equivalent — remap it as a grow-in *up to* 100 instead (e.g. `88` → `100`), never as `100` → `>100`. |
| `rotate` | `angle` | unbounded signed degrees, not clamped to ±360 |
| `fade` | `opacity` | 0.0–1.0 (a fraction — unlike `scale`'s 0–100, and unlike a layer's own `properties.opacity`, which is 0–100) |
| `audioFade` | `volumeEffect` | 0.0–1.0 |
| `textEffect` | `typewriting` (0 or 100 only) + `textEffectType` (`classic`, `softFade`, `focusIn`, `riseUp`, `riseDown`) | — |

Each layer can have **at most one tween of each type** (two `fade` tweens on one layer
fail with `invalid_payload`) and each tween needs at least 2 keyframes — chain several
keyframes inside one tween instead. Only a numeric keyframe `data.time` is enforced;
`data.type` (`start`/`middle`/`end`) and `ease` are the normal shape but optional.

GSAP ease strings carry over directly into `ease`, **including parametrized ones**
(`"back.out(1.7)"`, `"elastic.out(1, 0.5)"` both validate and render correctly).

**Multi-format and `slide` tweens don't mix safely.** A layer's `tweens` are declared
*once*, not per format, but a `slide` tween's `attr.left`/`attr.top` are absolute
canvas pixels — and if that layer's `layout` differs by format (the normal case for a
multi-format design), those absolute values can only be correct for one format's box.
The import docs now confirm it: the single `animation` block is applied to every
format as is, so `slide` offsets (and a `scale` tween's origin) are the same pixels in
every format — they are not reinterpreted per format. So on a multi-format animated
design, restrict `slide` to a layer whose `layout` box is identical across every
format, and use rotate/scale/fade for anything whose position varies by format (none
of those three reference a position). Per-format slide distances can only be adjusted
in the Abyssale editor after the import.

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
actually reaches opacity 0 — re-check its keyframes (and its `.from()`/`.to()` intent).

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
- **Linear/radial gradients** → use the native gradient string (see step 3) on the
  shape/button background or text color — no rasterizing needed.
- **`filter: blur()`, `backdrop-filter`, glassmorphism, gradients beyond what the
  native syntax supports** → don't approximate with a flat-color Abyssale shape, it
  reads as visibly wrong rather than simplified. If the effect is purely decorative,
  it's fine to drop it. If it matters to the design and you can run shell commands,
  rasterize just that portion with a headless browser
  (`google-chrome --headless=new --screenshot=out.png "file://<path>"`, with the
  layers you're keeping native hidden via injected `visibility: hidden` CSS first)
  and import the result as one `image` layer (hosted URL, or a data URI of at most
  100 KB). Without a shell, drop it and tell the user.
- **Per-character text stagger** → animate the whole text block as one unit; the
  stagger itself is lost, not approximated.
- **3D transforms (`rotateX`/`rotateY`)** → drop the 3D aspect, keep any accompanying
  2D motion (fade/slide) on the same element.
- **Multiple independently-animated pieces inside one SVG** → flatten the whole SVG
  to one rasterized `image` layer and apply a single simple tween to it (fade/scale
  in, say) rather than rasterizing each sub-element separately to preserve individual
  motion — that level of decomposition is not worth it for a best-effort pass.

## Measuring real positions (non-fixed-canvas layouts)

This needs shell access and headless Chrome. Without them, ask the user for a
fixed-size version of the banner (explicit container `width`/`height` and absolute
children), or estimate positions and tell the user plainly that they're estimates to
check in the Abyssale editor.

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
- Images: a hosted HTTPS URL is always the first choice. Inline base64 is only for
  small assets (at most 100 KB) and only when a tool produced it — never type base64
  yourself, a long string is easy to corrupt across tool calls. A bare local filename
  plus the upload flow (see `abyssale-import`) works only when you can run shell
  commands; never ask the user to run the upload commands.
