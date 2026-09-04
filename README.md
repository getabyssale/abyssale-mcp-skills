# Abyssale MCP Skills

Ready-made Claude Skills for Abyssale — turn designs into imports, variations, personalized batches, and print-ready PDFs.

These skills drive the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) through complete, tested workflows — install one and describe what you want in plain language instead of chaining tool calls yourself.

## Skills

| Skill | What it does |
|---|---|
| [`abyssale-import`](./abyssale-import) | Import a design into Abyssale from a JSON payload, including uploading local image assets |
| [`figma-to-abyssale`](./figma-to-abyssale) | Recreate a Figma design as a native, fully-editable Abyssale design |
| [`html-to-abyssale`](./html-to-abyssale) | Recreate an HTML/CSS banner (HTML5 ad, email banner, design-tool export) as a native Abyssale design, including best-effort CSS/GSAP animation |
| [`abyssale-variations`](./abyssale-variations) | Generate multiple format/size/language variations or a video from one design |
| [`abyssale-personalize`](./abyssale-personalize) | Bulk-generate personalized creatives from a dataset — one visual per row |
| [`abyssale-print-catalog`](./abyssale-print-catalog) | Generate a print-ready multi-page PDF catalog, brochure or lookbook |

## Where these work

These are [Claude Skills](https://www.anthropic.com/news/skills) — Claude Code and
claude.ai discover and invoke them automatically (by matching your request against
each skill's description) once installed. Other AI coding agents (Codex, Cursor,
etc.) don't have that same auto-discovery mechanism; a `SKILL.md` file still works
with them as plain reference material, but you'd point the agent at it yourself
(e.g. "read `html-to-abyssale/SKILL.md` before doing this") rather than relying on
it being picked automatically.

Either way, a skill only teaches a client *how* to use the Abyssale MCP tools — it
doesn't connect anything itself. See "Install" below.

## Install

Copy the skill folder(s) you want into your client's skills directory (e.g. `.claude/skills/` for Claude Code), or install via your client's plugin/marketplace mechanism if it supports one.

Each skill assumes the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) is already connected — see that page for setup instructions.
