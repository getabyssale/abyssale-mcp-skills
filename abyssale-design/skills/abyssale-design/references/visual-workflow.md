# Design, fidelity, and adaptation

This is a design method, not an API schema. Scale it to the request: a local correction does not need a new direction, mockup, and complete extraction. The composition choices below are recommendations.

## Frame the request without prescribing a style

Distinguish supplied or required elements (text, brand, data, subject, format, editability) from what remains to be designed. Choose dimensions from the brief and channel; neither a set of six formats nor a primary square format is required.

Define a short intent: what should be perceived first, tone, hierarchy, relative visual masses, palette, typography, and media roles. Add a CTA only when it serves the objective. Editorial, abstract, event, or informational work may follow a different logic from product advertising.

| Situation | Suitable method |
|---|---|
| New direction exploring photography, illustration, or atmosphere | Raster mockup with the runtime's image path (see the runtime reference), then decomposition |
| Typographic poster, data grid, precise vector composition | Direct native or vector construction; raster generation only for a useful asset |
| Faithful reference or restrictive brand guidelines | Analyze invariants, reuse assets, reconstruct without reinventing everything |
| Existing design to correct | Diagnose the defect in the render and structure; preserve unrelated decisions |
| Content variants or many adaptations | Define variables and composition rules before multiplying outputs |
| Animation, video, or multiple pages | Also design sequence, rhythm, and continuity across states/pages |

### Brand research and creative feedback

For a brand supplied by URL, examine relevant official assets: logo, palette, typography, iconography, images, and verifiable messages. Open candidate images; a filename or thumbnail does not validate their subject or quality. Keep exact assets when identity matters.

For past campaigns, note visual conventions and dates without automatically reusing an expired offer, claim, or ambassador. References inform the requested direction; they do not authorize changing the brief.

For feedback such as "too restrained," "too light," or "not faithful," identify the perceptible cause: composition, headline area, weight, contrast, angles, or image integration. Adjust that factor; do not add random decoration or turn project-specific feedback into a mandatory style.

## Mockups and creative freedom

Explore a raster direction when it improves the result, using the image path of the runtime driving the skill: native ImageGen in Codex ([runtime-codex.md](runtime-codex.md)); Abyssale's own AI through a scratch design, vector mockups or supplied assets in Claude ([runtime-claude.md](runtime-claude.md)). Inspect local files before editing and specify their roles (reference, editing target, asset to insert). If no image path is available, proceed with existing assets where possible and explain the limitation.

Describe the result in the prompt, not the future JSON. Do not default to restricting the mockup to Abyssale catalog fonts or requiring a font specimen before generation. Use a specific font when the brief requires it, while checking its raster reproduction.

Adapt this outline without artificially filling every category:

```text
Use and medium: [objective, audience, dimensions, output].
Exact content: [required text and data, without additions].
References: [role of each image and invariants].
Direction: [tone, composition, hierarchy, palette, typographic intent].
Spatial relationships: [desired overlaps or integration, if relevant].
Preserve: [product/character/logo identity, details, constraints].
```

Inspect the proposal at delivery size: readability, accuracy, subject, hierarchy, and graphic impact. Correct the direction if necessary, then use that version as the reconstruction target. Do not generate an entirely new mockup merely to fix alignment or import problems.

### Fidelity to the selected mockup

Identify invariants **in the visual**, not just the prompt:

- Text silhouette, width, weight, slant, and relative size; colors, punctuation, and emphasis.
- Block positions, axes, spacing, and text/image ratio.
- Subject, illustration details, characteristic shapes and motifs.
- Front/back relationships, continuity, lighting, and edges.

Absolute values may change with the ratio, but the intent should remain recognizable. Colored punctuation and small details contribute to identity; include them in comparisons. Supplied copy takes precedence over mockup text errors.

A correction or reproduction preserves these invariants. An explicit request for a new direction allows reconsidering them.

## Depth and overlap analysis

After selecting the direction and **before extraction**, analyze depth when the composition contains intersections. For a visual without complex overlaps, a simple element list and stacking order suffice; no fixed number of planes is required.

1. **Visual planes.** Identify distant scenery, intermediate planes, subjects, and nearby elements. A "background photo" may contain foliage, props, or walls that must appear in front of an added shape.
2. **Intersections.** At significant contacts, record what covers what and where motifs disappear and reappear. Include text, shadows, transparency, and soft edges when relevant to integration. Distinguish observed relationships from reconstruction choices.
3. **Operations.** Derive which assets to keep together, extract, or reconstruct, and their placement coordinates.

Compact record: **area → in front / behind → asset and operation → behavior by format**.

| Observed need | Decision |
|---|---|
| Elements stay together with nothing inserted between them | Keep one shared asset; avoid unnecessary cutouts |
| A shape passes between scenery and foliage/prop/subject | Extract that nearby plane with transparency and put the shape underneath |
| Movement reveals an obscured area | Reconstruct only the useful portion, with enough margin for planned crops |
| A ribbon alternates in front and behind | Distribute portions of a coherent path across planes; hide joins |
| Several extractions must coincide | Preserve shared canvas, scale, position, rotation, and crop; avoid automatic recentering |
| A new ratio changes the layout | Recompose planes while preserving characteristic relationships |

The main subject is not the only potential cutout: isolating a cyclist does not resolve a shape meant to pass behind a bush. Stacking order depends on the scene, not a "background → motif → character" recipe.

Preserve proportions, lighting, angles, and edge softness. If an element remains in the source beneath its extraction, check for doubled edges; clean the source if it must move independently. Generative processing may reinterpret the asset: check fidelity, position, and actual alpha before importing, without promising pixel-identical extraction.

## Produce assets and choose layers

The complete mockup is not the final background by default. Choose separation according to fidelity and expected edits, not an arbitrary target layer count.

- Keep variable content in Text and simple components in appropriate native types.
- Reuse exact logos and assets. For selected photographs or illustrations, start from that reference rather than an independently generated scene.
- Preserve defining visual details (characteristic motifs, props, useful shadows), either together or on separate planes according to their role.
- Keep photographs, textures, and complex illustrations in Image; an image needing no insertions can remain whole.
- For a highly stylized title with no close font, apply the [SVG exception](native-layers.md#highly-stylized-svg-lettering), including its editing limitation.
- Keep final assets in the workspace; uploads follow the [MCP contract](import-and-mcp.md#providing-assets-and-tracking-imports).

For extraction, describe what to keep, what to remove, the transparent or reconstructed background, and the spatial coordinates to preserve. Check alpha and edges against the intended background. A transformation that alters an exact product cannot be presented as a faithful cutout.

### Decorative shapes independent of photography

Prefer native geometry when sufficient: a rounded rectangle for a logo capsule, a circle for a badge. Reserve SVG for shapes or effects that available primitives cannot faithfully reproduce; appearing in a raster mockup alone does not justify exporting a shape as an image.

If a cutout, fill, wave, or path must vary independently of the crop, separate it from the photograph. Use Shape for suitable native geometry, or a standalone path-based SVG in an Image layer for a custom shape. Do not wrap the entire photograph in an SVG and claim decomposition.

Keep motifs geometrically complete beneath occluding elements. Drawing only visible pieces or trimming a path along a silhouette creates breaks when layers move. For alternating front/back passages, keep a shared trajectory and enough overlap at joins.

Choose position, scale, variant, crop, and visibility independently of the photograph for each format. Adapt or simplify structural motifs instead of hiding them for convenience; hide only dispensable elements or when explicitly constrained. Check transparency, color, proportions, and clipping in the render.

### Simple backgrounds and atmospheric textures

Analyze the base color separately from visible texture: smoke, grain, irregular light, diffuse glow. Supported solid fills and simple gradients belong in the native format background; do not create a full-frame image or SVG solely for this role. Consult [native properties](native-layers.md#choosing-the-right-type) for the channel and medium.

A characteristic mockup texture should not become a smooth gradient for convenience. When it must move or crop independently, extract or reconstruct it from the reference with the runtime's image tool (ImageGen in Codex, Abyssale AI via a scratch design in Claude), removing text, subjects, and motifs already in other layers. Preserve genuinely gradual alpha, soft edges, and grain, then composite the Image over the appropriate native background color. Black is an artistic option, not a general rule; do not claim a black base as fact unless the reference demonstrates it.

Check actual transparency and rendering over the chosen color: no opaque rectangle, fringe, or edge discontinuity. Adapt position, scale, crop, and opacity to each ratio to retain texture without impairing readability. Cover cropping may work; verify instead of applying it universally. When replacing a background, remove the obsolete layer and preserve other elements.

## Prepare composition by format

Record decisions in a design sheet or construction source: stable name, role, type, content/asset, geometry, rotation, order, styles, alignments, and variations. Do not copy these categories as API properties; use the selected type's contract.

### Choose alignment by ratio

Distinguish layout position, content alignment, and Button visible-box position. A centered block can still contain left-aligned text. Technical values and distinctions are in [native-layers.md](native-layers.md#alignment-values).

Evaluate a centered composition for narrow formats; a side axis may work better in columns. Follow the reference and brief: neither centering nor left alignment is automatic.

Recompose areas, line breaks, relative sizes, crops, and angles for each ratio. Uniform scaling suffices only if aspect ratio and readability remain suitable. Distinguish block rotation, italics, and distortion; see [rotation](native-layers.md#block-rotation-and-italics).

For small formats, simplify permitted secondary information and adapt motifs before shrinking every text excessively. Do not remove mandatory information to make the composition fit.

### Templates and variable content

Identify what changes: text, language, figures, prices, images, item counts. Anticipate representative short/long cases, extreme values, and different crops. Keep content editable and test line breaks, autofit, alignments, and component behavior.

Check required glyphs, accents, and writing systems with available fonts; do not assume variants, emoji, or languages render like the reference. A different reading direction may require a different composition. Avoid outlining dynamic content; explicitly resolve any necessary tradeoff.

## Checking the result

After [validation, import, and saving](import-and-mcp.md), inspect actual outputs without overrides used merely to hide a saved-design defect.

| Check | Inspect |
|---|---|
| Content | Exact, complete text, correct data, preserved emphasis and punctuation |
| Graphic impact | Hierarchy, headline mass and silhouette, subject presence, diagonals, space, and contrast |
| Typography and layout | Weights, actual rendered sizes, spacing, alignment, rotation, and margins |
| Assets | Sharpness, identity, proportions, framing, transparency, and edge quality |
| Overlaps | Natural edges, motif disappearance/reappearance, no halos, holes, breaks, or duplication |
| Reuse | Appropriate layer types, required editing, and representative variations |
| Adaptations | Recognizable intent in every format, including at delivery size |

Compare the main render with the reference, then each adaptation with its invariants. Inspect enlarged contact areas when integration is fragile. A warning-free export, correct layer count, or technically fitted text does not prove visual quality.

### Medium-specific checks

- **Static:** inspect every delivered format at delivery size and in detail when needed.
- **Animation/video:** inspect playback, duration, readability during movement, transitions, overlaps over time, start/end/loop, and intended sound. A snapshot does not validate the sequence. Follow [specialized layers](specialized-layers.md).
- **Print:** verify physical dimensions, units, effective resolution, bleed, safe area, and color settings against supplied specifications. Do not invent printer requirements.
- **Multipage:** check every page, order, continuity, content, and recurring elements. Verify the `printer_multipage` type and contract before serialization.

Correct the observed cause and repeat affected checks. If an effect is available only in the editor, finalize there and verify persistence; if no compatible route exists, explain the tradeoff instead of inventing a property. Once criteria are met, deliver without arbitrary new exploration.

## Delivery and limitations

Provide the verified link, actual preview, formats, and remaining observed limitations: substitution, frozen content, generative retouching, or unverified medium. Keep source JSON and assets useful for reproduction.

For a new creation, remove construction residues hidden everywhere and no longer useful unless there is an explicit reason to keep them. For targeted corrections, limit cleanup to layers created or made obsolete by the intervention; do not delete unrelated preexisting layers. Layers needed in some adaptations remain relevant.

The [examples](examples.md) are small control designs; recheck the capability and the render the actual request depends on.
