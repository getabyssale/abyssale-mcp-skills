---
name: abyssale-import
description: Import a design into Abyssale from a JSON payload (e.g. exported from Figma, hand-built, or round-tripped from an existing Abyssale design), including uploading local image assets. Use when the user wants to create an Abyssale template/design programmatically instead of building it by hand in the editor, or asks to "import a design into Abyssale", "turn this JSON/Figma export into an Abyssale template", or "recreate this design in Abyssale".
---

# Import a design into Abyssale

Abyssale designs can be created programmatically from a JSON payload — no designer has to build the template by hand. This skill drives that flow end to end, including the case where the design references image files that live on the user's own machine rather than an already-hosted URL.

## Prerequisites

The Abyssale MCP server must be connected and authorized with the `import:design` scope (and `read:design` if you'll look up an existing design's format first). If a tool call fails with an insufficient-scope error, tell the user to reconnect the Abyssale MCP client and grant that scope.

## Steps

1. **Get the target JSON.**
   - If the user already has a JSON payload (e.g. a Figma export or something hand-written), use it directly.
   - If they're unsure of the expected shape, call `get_design_import_example` for a valid example payload, or `get_design_as_import` on an existing Abyssale design to get its JSON as a starting point (round-tripping).
   - If they want a *specific existing design's* structure (layer types, fields) as reference rather than a ready-to-edit import payload, call `get_design_format`.

2. **Submit the import.** Call `import_design_from_json` with the payload.

3. **Handle the response:**
   - Finalized immediately → the design is created. Report its identifier back to the user.
   - `WAITING_FOR_VALIDATION` → the payload referenced at least one image by a bare filename instead of a hosted URL. Abyssale minted one presigned upload per referenced image; the response (or the next status poll) includes ready-to-run `curl` commands.

4. **When there are pending uploads:**
   - Show the user the exact `curl` commands verbatim — don't paraphrase them or turn them into JSON. **They have to run these themselves, on their own machine**: the referenced files live on their filesystem and upload directly from there, so you cannot run the upload on their behalf.
   - Don't reorder the command's fields and don't move the file field from the end — the upload will be rejected if the field order doesn't match what was signed.

5. **Poll for completion.** Once the user says they've run the upload commands (or periodically if they're slow to get to it), call `check_design_import_status` again. This call also re-validates the import on every call, so polling early is safe and free — it won't fail the import.

6. **Terminal failure case.** If status comes back failed with no `expected` field alongside a missing-assets style error, the upload window has expired (it's time-limited) and restarting the upload won't fix it. Tell the user to restart the import from step 2.

## Notes

- Never invent or guess the JSON structure — always source it from `get_design_import_example` / `get_design_as_import` / `get_design_format` rather than assuming a shape.
- There is no separate "validate" tool — `check_design_import_status` both checks and validates in one call.
