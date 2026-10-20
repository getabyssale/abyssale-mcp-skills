# Specialized layers and animation

Read this before creating or editing Rating, Illustration, Video, Audio, or animation. Import and generation contracts are distinct; settings not available through the MCP are listed in [MCP limits](mcp-limits.md).

## Type universe

Choose components according to need, without adding animation, ratings, or media by default. For animation, establish states, reading durations, transitions, temporal stacking order, and intended sound before tweens; a satisfactory static composition alone does not define a sequence. [Medium-specific checks](visual-workflow.md#medium-specific-checks) complement these contracts.

Creation accepts **11 types**: `text`, `image`, `logo`, `shape`, `button`, `rating`, `qrcode`, `group`, `illustration`, plus `video` and `audio` for `animated` designs. [Create Import](https://developers.abyssale.com/rest-api/designs/import/create), [Exact schema](https://developers.abyssale.com/api-reference/createDesignImport).

The generation override catalog describes **10 components**: the same set without `group`. Background is addressed separately by `root.background_color`. `group` carries structure; `container` is a technical read envelope, not a layer to create. [Element Properties](https://developers.abyssale.com/rest-api/generation/element-properties), [Layers Overview](https://developers.abyssale.com/rest-api/layers-overview).

`get_design` may also list a **`code`** layer (Custom HTML, animated designs only), with no overridable attributes. It isn't importable or exported — don't invent `type: code` or `type: html`. Don't confuse it with QR Code.

## Options matrix

| Type | JSON creation | Render override |
|---|---|---|
| **Rating** | `rating_score` 0–100, `star_dimension` 4–400, `star_margin` 0–100, `star_color`, `background_color`, `opacity`, `rotation`, structured `shadow`. | Score, size, spacing, both colors, flattened shadow (`shadow_*`), common `hidden`. |
| **Illustration** | Required `illustration_type`, `illustration_file`; `illustration_properties.primary_color`, `alignment`, `opacity`, `rotation`. | Library, file, `illustration_properties`, `alignment`, `opacity`; compare common properties with `get_design`. |
| **Video** | Required `src`, `muted`, `opacity`, layout and timing. MP4/WebM; presigned upload 100 MB. Videos contiguous in `layers[]`, never group members. | `video_url` (MP4, 100 MB), `audio_muted` 0–1, `hidden`. Trim through URL `trim_start`/`trim_end`. |
| **Audio** | `src`, `muted`, `max_volume` 0–1; MP3/WAV, presigned upload 25 MB. `muted:true` forces zero volume. Needs a placeholder `layout` (e.g. 1×1). Group allowed. | MP4 output: `audio_url` or `audio_encoded` (URL takes precedence), `max_volume` 0–1, `speed` 0.25–2. |
| **Code** | Not importable; not exported. | None. |

Sources: [Rating help](https://help.abyssale.com/en/articles/92197-add-a-rating-layer), [Rating import](https://developers.abyssale.com/rest-api/designs/import/layer-types/rating), [Rating generation](https://developers.abyssale.com/rest-api/generation/element-properties/rating); [Illustration help](https://help.abyssale.com/en/articles/92207-add-an-illustration-or-icon-that-is-not-an-image-rating-text-or-shape), [Illustration import](https://developers.abyssale.com/rest-api/designs/import/layer-types/illustration), [Illustration generation](https://developers.abyssale.com/rest-api/generation/element-properties/illustration); [Video import](https://developers.abyssale.com/rest-api/designs/import/layer-types/video), [Video generation](https://developers.abyssale.com/rest-api/generation/element-properties/video); [Audio import](https://developers.abyssale.com/rest-api/designs/import/layer-types/audio), [Audio generation](https://developers.abyssale.com/rest-api/generation/element-properties/audio).

## Notes per type

- **Rating:** fixed row of **5 stars**; score is a percentage (64 → 3.2 stars). No documented star-count control. Use solid colors for both rating colors and verify rendering.
- **Illustration:** `undraw`, `feather`, `material_icons`, `twemoji`. The first three allow primary color. Twemoji keeps its own colors. Don't generalize one library's properties to the others.
- **Video:** fitting, alignment and masks aren't import properties. Don't transfer `fitting_type`, `alignment`, masks, volume or speed between contracts.
- **Audio:** `speed` is a generation property, not an import property — don't add it to import payloads.

## Animation: controls and contract

Animations: Fade, Scale, Slide, Rotate on visual layers (Video: no Rotate), plus text-only reveals through `textEffect` (`classic`, `softFade`, `focusIn`, `riseUp`, `riseDown`) and `audioFade` for audio/video. Groups can't be animated directly. Each keyframe can take an `ease` (GSAP names such as `power2.out`). [Animation help](https://help.abyssale.com/en/articles/562372-creating-and-customizing-animated-designs-in-abyssale), [Easing help](https://help.abyssale.com/en/articles/401830-understanding-and-use-easing).

Import `animation.duration` is 1–60 s; `screenshot_at_s` selects the fallback instant. Each layer may have `start_at_s`, `end_at_s`, `tweens`. Types: `slide`, `fade`, `scale`, `rotate`, `audioFade`, `textEffect`. One tween per type, at least two keyframes. `audioFade` is for audio/video; `textEffect` for text. Groups allow timing, but no tweens or animated auto-layout.

Preserve exported tweens: `opacity` and `volumeEffect` are 0–1 fractions; `scale` is a percentage; `typewriting` accepts only 0/100. Import shares a single animation block across formats, so slide offsets and scale origins are the same in every format. Keyframe `data.time` must fall within the layer's window. [Animated Imports](https://developers.abyssale.com/rest-api/designs/import/animated).

## Verification

1. After an import, compare `get_design_format` reads with your payload; when re-importing a corrected copy, read every export warning ([MCP limits](mcp-limits.md)).
2. Inspect the actual animation playback, not only `screenshot_at_s`: reading time, entrances/exits, moving overlaps, loop join if intended, synchronization and sound level. For HTML5, also test the exported output and requested interactions when the environment permits. Say what you couldn't verify.
