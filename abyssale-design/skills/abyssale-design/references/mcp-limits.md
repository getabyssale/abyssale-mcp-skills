# Not available through the MCP

These settings exist in Abyssale designs but can't be set through `import_design_from_json`, and the export (`get_design_as_import`) doesn't carry them. Don't invent a property for them: tell the user plainly that the setting isn't available through the MCP.

| Setting | Notes |
|---|---|
| Text: Max lines, Balance text visually, Truncate text | No import property (`max_lines` and `text_harmony` are rejected). Use `min_font_size` and a realistic box to control fitting |
| Text: `min_font_size` on an existing design | Can be set at import, but isn't returned by the export — keep it in your source payload and re-apply it when you re-import a corrected version |
| Custom HTML code layer (animated designs) | Not an import type; `get_design` may list it as `code`, with no overridable attributes |
| Video: fitting, alignment, masks | Not import properties for video layers |

Before re-importing a corrected copy of an existing design, read every export warning (`property_not_representable`): the listed settings are lost in the new version. Tell the user which ones, before they approve the new version.
