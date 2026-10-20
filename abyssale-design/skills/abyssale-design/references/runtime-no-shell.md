# Runtime: no shell (ChatGPT web/app, claude.ai without code execution, other chat clients)

Read this when the environment has the Abyssale connector but **no local shell**. Check which tools you actually have before starting — web access, a code sandbox, image viewing, image generation — and use the matching path below. The Abyssale contract and the rest of the skill don't change; only how files move and how you inspect results.

## What changes

| Step | With a shell (Codex, Claude Code, Cowork) | Without a shell |
|---|---|---|
| Brand research from a URL | Download the page and its CSS, extract tokens, logo and fonts | Use web browsing/search if available to read the site; otherwise ask the user for brand colors, fonts, logo URL and official taglines. Never guess brand values |
| Build the payload | Generate it with a script | Write the JSON directly, starting from `get_design_import_example` or `get_design_as_import`; keep it compact (fewer formats per import if needed) and always run `validate_only` first |
| Images | ImageGen (Codex), local files uploaded with `upload_command` | Public HTTPS URLs (brand site, user-supplied links), Abyssale-hosted URLs, Abyssale's own AI through the scratch design ([runtime-claude.md](runtime-claude.md#scratch-design-ai-pipeline)), native illustration layers. A chat-generated image (e.g. ChatGPT image generation) has no URL Abyssale can fetch: use it only if a code sandbox can read its bytes into a data URI of at most 100 KB — never type base64 yourself |
| Local files | Filename `src` + run `upload_command` yourself | Not possible: ask the user for a hosted link. Never hand the user an upload command |
| Mockups, SVG previews, contact sheet | Render locally | Skip local rendering; go straight to a first native import, and use generation overrides to test variants |
| Visual QA | Download renders and view them | If you can view images, inspect the returned previews/URLs format by format. If not, give the user the links, report what you checked technically (status, `errors`, warnings, `get_design_format` values) and say plainly that visual QA wasn't done |

## Rules that stay the same

- Never simulate an import or a render; if the connector is missing, say so.
- Every re-import creates a new design: plan the layout before the first import, iterate with generation overrides, and offer to delete drafts at the end (`delete_design`, after confirmation).
- Report substitutions (fonts, assets), AI-generated images and credits used.
