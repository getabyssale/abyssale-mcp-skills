# Abyssale MCP Skills

Ready-made skills for Abyssale — create designs and turn them into imports, variations, personalized batches, and print-ready PDFs.

These skills drive the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) through complete, tested workflows — install one and describe what you want in plain language instead of chaining tool calls yourself.

## Skills

| Skill | What it does |
|---|---|
| [`abyssale-design`](./abyssale-design) | Create, adapt or fix editable Abyssale designs (static, animated, print, multipage) from a brief, a brand URL, a reference or an existing design — with brand research, AI images and visual QA |
| [`abyssale-import`](./abyssale-import) | Import a design into Abyssale from a JSON payload, with images passed as URLs |
| [`figma-to-abyssale`](./figma-to-abyssale) | Recreate a Figma design as a native, fully-editable Abyssale design |
| [`html-to-abyssale`](./html-to-abyssale) | Recreate an HTML/CSS banner (HTML5 ad, email banner, design-tool export) as a native Abyssale design, including best-effort CSS/GSAP animation |
| [`abyssale-variations`](./abyssale-variations) | Generate outputs from one design in several of its formats, languages or text variations, or a video/GIF/HTML5 from an animated design |
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

**Claude Code, via the plugin marketplace** (recommended — this repo ships a
`.claude-plugin/marketplace.json`):

```
/plugin marketplace add getabyssale/abyssale-mcp-skills
/plugin install html-to-abyssale@abyssale-mcp-skills
```

Repeat the `install` line for any other skill from the table above (`abyssale-design`,
`abyssale-import`, `figma-to-abyssale`, `abyssale-variations`, `abyssale-personalize`,
`abyssale-print-catalog`).

**Manually, anywhere else**: copy the skill folder(s) you want into your client's
skills directory (e.g. `.claude/skills/` for Claude Code without the marketplace flow).
For a client without Claude's Skill auto-discovery (Codex, Cursor, etc.), a `SKILL.md`
still works as plain reference material — point the agent at it yourself rather than
relying on it being picked automatically.

**Any other agent, via the `skills` CLI**: [`skills`](https://github.com/vercel-labs/skills)
(Vercel Labs, not an Abyssale/Anthropic tool — third-party, mentioned here as a convenience,
not maintained by us) auto-discovers `SKILL.md` files in any public GitHub repo across 20+
agents. No submission or registration needed — it works directly against this repo:

```bash
npx skills add getabyssale/abyssale-mcp-skills --skill html-to-abyssale
```

Swap `--skill` for any other skill name from the table above, or drop the flag to install all
seven. Needs Node ≥22.20 — on an older Node this fails with a `styleText` import error; switch
with `nvm install 22 && nvm use 22` first if you hit that.

Each skill assumes the [Abyssale MCP server](https://developers.abyssale.com/mcp/overview) is already connected — see that page for setup instructions.
