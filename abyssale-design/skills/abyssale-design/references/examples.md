# Examples

Small payloads that import and render as described. They are control designs, not templates to copy.

| File | What it shows |
|---|---|
| [native-components-multiformat.json](examples/native-components-multiformat.json) | Two formats (`square-1200`, `portrait-900`) sharing one layer set: a decorative Shape, a Text headline, a rich-text Text layer (color, weight, underline, strikethrough, highlight, relative size, superscript/subscript, accents) and three Buttons (badge, label, multiline callout) arranged differently per format |
| [auto-layout-min-font-control.json](examples/auto-layout-min-font-control.json) | A vertical auto-layout group with a headline and a button, and a headline with `min_font_size` so a longer copy shrinks to fit. The button is last in `layers[]`, so it renders above the headline — see the auto-layout order note in [capabilities.md](capabilities.md#auto-layout-groups) |
| [scratch-ai-design.json](examples/scratch-ai-design.json) | The scratch design used to run Abyssale's image AI ([runtime-claude.md](runtime-claude.md#scratch-design-ai-pipeline)) |

Before using one:
- set `target.project_uuid` to a real project of the user's (`list_projects`), and resolve every `<FONT_ID>` with `list_fonts`;
- change the `name`;
- validate with `validate_only: true`, then import and render.

## Sources

[Help Center](https://help.abyssale.com/en/), [Generation properties](https://developers.abyssale.com/rest-api/generation/element-properties), [Import and examples](https://developers.abyssale.com/rest-api/designs/import/examples), [Export](https://developers.abyssale.com/rest-api/designs/as-import).
