---
name: abyssale-import
description: Import a design into Abyssale from a JSON payload in Abyssale's import format (hand-built, adapted from an official example, or round-tripped from an existing Abyssale design), with its images passed as URLs. Use when the user wants to create an Abyssale template/design programmatically instead of building it by hand in the editor, or asks to "import a design into Abyssale", "turn this JSON into an Abyssale template", "duplicate this design into another project", or "recreate this design in Abyssale".
---

# Import a design into Abyssale

Abyssale designs can be created programmatically from a JSON payload — no designer has to build the template by hand. This skill drives that flow end to end, with images passed as URLs.

## Prerequisites

The Abyssale MCP server must be connected and authorized with the `import:design` scope, plus `read:design` (examples and existing designs) and `read:project` (to pick the target project) — `write:project` only if you create a new project. Importing requires an Admin, Manager or Designer role; creating a project requires Admin or Manager. If a tool call fails with an insufficient-scope error, tell the user to reconnect the Abyssale MCP client and grant that scope.

## Steps

1. **Build the payload.**
   - **Start from a valid shape — never invent the JSON structure.** If the user already has a payload in Abyssale's import format, use it. A raw export from another tool is not in this format and can't be imported as is. Otherwise call `get_design_import_example` for an official example (`static`, `static-multi-format`, `animated`, `animated-video`, `printer`, `printer-multipage`), or `get_design_as_import` on an existing Abyssale design to get its JSON (round-tripping — e.g. to duplicate a design: change its `name` and target, then import).
   - **Choose where it lands.** Set `target.project_uuid` to one of the user's projects: call `list_projects` and ask which one to use, or `create_project` if they want a new one. Never leave the `YOUR_PROJECT_UUID` placeholder from the examples. Use `target.kind: "workspace_template"` (no project) only if the user asks for a template available to the whole workspace.
   - **Multi-format designs:** every layer's `layout` and `properties` must be keyed by format name (a flat shape is only valid with a single format) — start from the `static-multi-format` example.
   - **Images as URLs.** Every image should be a public HTTPS URL — this works in every client. If the user has no URL for an image, ask for one. Only if that isn't possible and you can read the file's bytes with a tool (code sandbox, shell), inline it as a `data:image/...;base64,` URI of at most 100 KB — never type base64 yourself — and tell the user if you recompressed it and how much quality was lost (e.g. "320 KB → 17 KB").
   - **Fonts** for text layers: use the `id` returned by `list_fonts`.

2. **Submit the import.** Call `import_design_from_json` with the payload.
   - For a large or hand-built payload, first call it with `validate_only: true`: it checks the payload without creating anything. It answers `status: DONE` with an empty `import_id` and no `result` — nothing was created, so run the real import afterwards and never report a design ID from a validation call. It doesn't check fonts, image URLs or the project.
   - `strict: true` makes the import fail when a text doesn't fit its box. By default the design is still created with a `text_fit_failed` warning.
   - The call waits up to ~60 seconds and usually returns the final status directly. Pass `wait_for_result: false` to get the `import_id` back immediately.

3. **Handle the response** (from the import call, or later from `check_design_import_status` with the returned `import_id`):
   - `DONE` (`is_finalized: true`) → the design is created. Report `result.uuid` to the user, and mention any `warnings` (e.g. `text_fit_failed`, `font_weight_adjusted`).
   - `FAILED` (`is_finalized: true`) → terminal. Show the returned `error`, fix it, and start a new import.
   - `QUEUED` (all inputs received, waiting to start) or `PROCESSING` (design being built) → normal in-progress states, not errors. Check again later with `check_design_import_status`, not in a tight loop.
   - `WAITING_FOR_VALIDATION` → only happens when the payload named a local file instead of a URL. Prefer restarting the import with a URL. Exception, only if you can run shell commands on the machine holding the files (e.g. Claude Code, Codex): the first import response contains one upload command per image layer — they are never repeated by status checks, so keep them. Run each one yourself, unchanged except for the real file path (don't reorder its fields), then call `check_design_import_status` to start processing. Never ask the user to run them. They expire one hour after the import was created — restart an expired import instead of reusing them. (Claude.ai needs the Abyssale upload bucket on the workspace's domain allowlist — see the MCP overview docs.)

## Notes

- Two different checks: `validate_only` checks the payload; `check_design_import_status` reports progress and, after local uploads, confirms the files landed and starts processing.
- Every format of an image layer that uses a local file must name the same file; use HTTPS URLs if formats need different images.
