# Abyssale MCP Skills

Ready-made Claude Skills for Abyssale — turn designs into imports, variations, personalized batches, and print-ready PDFs.

These skills drive the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) through complete, tested workflows — install one and describe what you want in plain language instead of chaining tool calls yourself.

## Skills

| Skill | What it does |
|---|---|
| [`abyssale-import`](./abyssale-import) | Import a design into Abyssale from a JSON payload, including uploading local image assets |
| [`figma-to-abyssale`](./figma-to-abyssale) | Recreate a Figma design as a native, fully-editable Abyssale design |
| [`abyssale-variations`](./abyssale-variations) | Generate multiple format/size/language variations or a video from one design |
| [`abyssale-personalize`](./abyssale-personalize) | Bulk-generate personalized creatives from a dataset — one visual per row |
| [`abyssale-print-catalog`](./abyssale-print-catalog) | Generate a print-ready multi-page PDF catalog, brochure or lookbook |

## Install

Copy the skill folder(s) you want into your client's skills directory (e.g. `.claude/skills/` for Claude Code), or install via your client's plugin/marketplace mechanism if it supports one.

Each skill assumes the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) is already connected — see that page for setup instructions.
